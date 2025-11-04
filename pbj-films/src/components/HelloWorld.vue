<template>
  <v-container class="fill-height" max-width="900">
    <div>
      <v-img class="mb-4" height="150" src="@/assets/logo.png" />

      <div class="mb-8 text-center">
        <div class="text-body-2 font-weight-light mb-n1">Welcome to</div>
        <h1 class="text-h2 font-weight-bold">PBJ Films</h1>
      </div>

      <div class="actors-container">
        <h2>Actors with most appearances</h2>
        <div class="actors-list">
          <div v-for="actor in actors" :key="actor.id">
            <img
              v-if="actor.profile_path"
              :alt="actor.name"
              class="actor-image"
              :src="`https://image.tmdb.org/t/p/w185${actor.profile_path}`"
            >
            <div v-else class="actor-image-placeholder">No Image</div>
            <div class="actor-info">
              <div class="actor-header">
                <h3>{{ actor.name }}</h3>
                <button class="info-button" :title="`Show movies for ${actor.name}`" @click="toggleMovieList(actor.id)"> <svg
                  fill="none"
                  height="20"
                  stroke="currentColor"
                  stroke-width="2"
                  viewBox="0 0 24 24"
                  width="20"
                >
                  <circle cx="12" cy="12" r="10" />
                  <line x1="12" x2="12" y1="16" y2="12" />
                  <line x1="12" x2="12.01" y1="8" y2="8" />
                </svg></button>
              </div>
            </div>
            <p class="actor-details">
              {{ actor.known_for_department }} • {{ actor.appearances }} appearance{{ actor.appearances > 1 ? 's' : '' }}
            </p>
            <!-- Movies list (expanded) -->
            <transition name="slide">
              <div v-if="expandedActorId === actor.id" class="movies-list">
                <h4>Movies:</h4>
                <ul v-if="actor.movies && actor.movies.length > 0">
                  <li v-for="movie in actor.movies" :key="movie.id" class="movie-item">
                    <div class="movie-details">
                      <strong>{{ movie.title }}</strong>
                      <span class="movie-meta">
                        <span v-if="movie.character" class="character">
                          as {{ movie.character }}
                        </span>
                      </span>
                    </div>
                  </li>
                </ul>
                <ul v-else-if="actor.movie_ids && actor.movie_ids.length > 0">
                  <li v-for="movieId in actor.movie_ids" :key="movieId" class="movie-item">
                    Movie ID: {{ movieId }}
                  </li>
                </ul>
                <p v-else class="no-movies">No movies recorded.</p>
              </div>
            </transition>
          </div>
        </div>
      </div>

      <v-row>
        <v-col v-for="movie in movies" :key="movie.tmdb_id" cols="6">
          <v-card class="mx-auto" rounded="lg" :title="movie.title">
            <v-divider />
            <v-card-text>
              <span><i><b>Director:</b> {{ movie.credits.crew.find(person => person.job === "Director").name }}</i>
                <br></span>
              <span><i><b>Date watched:</b> {{ movie.date_watched }}</i> </span>
              <v-divider />

              <ul>
                <li v-for="person in movie.credits.cast.slice(0, 9)" :key="person.id">
                  {{ person.name }} <span class="text-gray-500">as</span> {{ person.character }}
                </li>
              </ul>
            </v-card-text>

          </v-card>
        </v-col>

      </v-row>

      <v-row>
        <div>

          <div v-if="status" class="status">{{ status }}</div>

          <div v-if="errors.length > 0" class="errors">
            <h3>Errors:</h3>
            <ul>
              <li v-for="(error, index) in errors" :key="index">{{ error }}</li>
            </ul>
          </div>
        </div>
      </v-row>

    </div>
  </v-container>
</template>

<script setup>

  import { onMounted, ref } from 'vue'
  import { supabase } from '../lib/supabaseClient'
  const actors = ref([])
  const movies = ref([])
  const expandedActorId = ref(null)

  onMounted(() => {
    loadActors()
    loadMovies()
    processMovies()
  })

  const isProcessing = ref(false)
  const status = ref('')
  const errors = ref([])

  async function processMovies () {
    isProcessing.value = true
    status.value = 'Fetching movies from database...'
    errors.value = []

    try {
      // Fetch all movies with credits from pbj-movies table
      const { data: movies, error: fetchError } = await supabase
        .from('pbj-movies')
        .select('tmdb_id, title, credits')
        .order('date_watched', { ascending: false })

      if (fetchError) throw fetchError

      if (!movies || movies.length === 0) {
        status.value = 'No movies found in database.'
        return
      }

      status.value = `Found ${movies.length} movies. Processing actors...`

      // First, get all existing actors to calculate new appearance counts
      const { data: existingActors } = await supabase
        .from('pbj_actors')
        .select('id, movies, movie_ids')

      const existingMap = new Map(
        existingActors?.map(a => [a.id, {
          movie_ids: a.movie_ids || [],
          movies: a.movies || [],
        }]) || [],
      )

      // Collect all actors from all movies
      const actorMap = new Map()

      for (const movie of movies) {
        const movieId = movie.tmdb_id
        const credits = movie.credits

        if (!credits || !credits.cast) {
          errors.value.push(`Movie ${movieId} has no cast data in credits`)
          continue
        }

        const cast = credits.cast

        if (!Array.isArray(cast)) {
          errors.value.push(`Movie ${movieId} has invalid cast data`)
          continue
        }

        for (const actor of cast) {
          const actorId = actor.id

          // Get existing data for this actor
          const existing = existingMap.get(actorId)

          if (!actorMap.has(actorId)) {
            // First time seeing this actor in current batch
            // Start with their existing movies
            const existingMovies = existing?.movies || []
            const existingMovieIds = existing?.movie_ids || []

            actorMap.set(actorId, {
              id: actorId,
              name: actor.name,
              original_name: actor.original_name,
              profile_path: actor.profile_path,
              known_for_department: actor.known_for_department,
              popularity: actor.popularity,
              movies: [...existingMovies],
              movie_ids: [...existingMovieIds],
            })
          }

          // Get current actor data from map
          const actorData = actorMap.get(actorId)

          // Check if using movies (JSONB) or movie_ids (array)
          if (actorData.movies === undefined) {
            // Using JSONB movies format
            const movieExists = actorData.movies.some(m => m.id === movieId)

            if (!movieExists) {
              const movieInfo = {
                id: movieId,
                title: movie.title,
                character: actor.character,
              }
              actorData.movies.push(movieInfo)
            }

            actorData.appearances = actorData.movies.length
          }
        }
      }

      const actors = Array.from(actorMap.values())

      status.value = `Processing ${actors.length} unique actors...`

      // Use upsert to insert or update in a single operation
      const { error: upsertError } = await supabase
        .from('pbj_actors')
        .upsert(actors, {
          onConflict: 'id',
          ignoreDuplicates: false,
        })

      if (upsertError) throw upsertError

      status.value = `Successfully processed ${actors.length} actors from ${movies.length} movies!`
    } catch (error) {
      console.error('Error processing movies:', error)
      errors.value.push(`Failed to process movies: ${error.message}`)
      status.value = 'Processing failed. See errors below.'
    } finally {
      isProcessing.value = false
    }
  }

  async function loadActors () {
    try {
      const { data, error } = await supabase.from('pbj_actors').select('*').limit(10).order('appearances', { ascending: false })
      if (error) throw error
      if (data) actors.value = data || []
    } catch (error) {
      console.error('Error loading actors:', error)
    }
  }

  async function loadMovies () {
    try {
      const { data, error } = await supabase.from('pbj-movies').select('*').limit(10).order('date_watched', { ascending: false })
      if (error) throw error
      if (data) movies.value = data || []
    } catch (error) {
      console.error('Error loading movies:', error)
    }
  }

  function toggleMovieList (actorId) {
    // Toggle: if already expanded, collapse it; otherwise expand it
    expandedActorId.value = expandedActorId.value === actorId ? null : actorId
  }

</script>

<style scoped>
.text-gray-500 {
  color: #6b7280;
  /* Tailwind gray-500 */
}

.actors-container {
  padding: 20px;
  max-width: 1200px;
  margin: 0 auto;
}

.actors-list {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
  gap: 20px;
  margin-top: 20px;
}

.actor-card {
  background: white;
  border: 1px solid #e0e0e0;
  border-radius: 8px;
  padding: 15px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  transition: box-shadow 0.2s;
}

.actor-card:hover {
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.15);
}

.actor-image {
  width: 100%;
  height: 200px;
  object-fit: cover;
  border-radius: 4px;
  margin-bottom: 10px;
}

.actor-image-placeholder {
  width: 100%;
  height: 200px;
  background: #f0f0f0;
  border-radius: 4px;
  display: flex;
  align-items: center;
  justify-content: center;
  color: #999;
  margin-bottom: 10px;
}

.actor-info {
  width: 100%;
}

.actor-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 10px;
  margin-bottom: 8px;
}

.actor-header h3 {
  margin: 0;
  font-size: 18px;
  flex: 1;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.info-button {
  background: #2196F3;
  color: white;
  border: none;
  border-radius: 50%;
  width: 32px;
  height: 32px;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  transition: background 0.2s;
  flex-shrink: 0;
}

.info-button:hover {
  background: #1976D2;
}

.info-button:active {
  transform: scale(0.95);
}

.actor-details {
  color: #666;
  font-size: 14px;
  margin: 0 0 10px 0;
}

.movies-list {
  margin-top: 15px;
  padding-top: 15px;
  border-top: 1px solid #e0e0e0;
}

.movies-list h4 {
  margin: 0 0 10px 0;
  font-size: 16px;
  color: #333;
}

.movies-list ul {
  list-style: none;
  padding: 0;
  margin: 0;
}

.movie-item {
  padding: 8px 0;
  border-bottom: 1px solid #f0f0f0;
}

.movie-item:last-child {
  border-bottom: none;
}

.movie-details {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.movie-details strong {
  color: #333;
  font-size: 14px;
}

.movie-meta {
  display: flex;
  gap: 8px;
  font-size: 12px;
  color: #666;
}

.release-date {
  color: #888;
}

.character {
  color: #2196F3;
  font-style: italic;
}

.no-movies {
  color: #999;
  font-style: italic;
  font-size: 14px;
}

/* Slide transition */
.slide-enter-active,
.slide-leave-active {
  transition: all 0.3s ease;
  max-height: 500px;
  overflow: hidden;
}

.slide-enter-from,
.slide-leave-to {
  max-height: 0;
  opacity: 0;
  transform: translateY(-10px);
}
</style>
