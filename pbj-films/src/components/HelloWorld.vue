<template>
  <v-container class="fill-height" max-width="900">
    <div>
      <v-img class="mb-4" height="150" src="@/assets/logo.png" />

      <div class="mb-8 text-center">
        <div class="text-body-2 font-weight-light mb-n1">Welcome to</div>
        <h1 class="text-h2 font-weight-bold">PBJ Films</h1>
      </div>

      <v-row cols="5">
        <v-list lines="one">
          <v-list-item v-for="actor in topActorsApps" :title="actor.name"
            :subtitle="actor.newAppearances"></v-list-item>
        </v-list>
      </v-row>

      <v-row>
        <v-col v-for="movie in movies" :key="movie.tmdb_id" cols="6">
          <v-card :title="movie.title" rounded="lg" class="mx-auto">
            <v-divider></v-divider>
            <v-card-text>
              <span><i><b>Director:</b> {{movie.credits.crew.find(person => person.job === "Director").name}}</i>
                <br></span>
              <span><i><b>Date watched:</b> {{ movie.date_watched }}</i> </span>
              <v-divider></v-divider>

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

          <div v-if="errors.length" class="errors">
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

import { ref, onMounted, computed } from 'vue'
import { supabase } from '../lib/supabaseClient'
const movies = ref([])
const actors = ref([])
const topActorsApps = ref([])

onMounted(() => {
  loadMovies()
  processMovies()
  topActors()
})


const loadMovies = async () => {

  let { data, error
  } = await supabase
    .from('pbj-movies')
    .select('*')

  console.log('data: ', data)
  if (data) movies.value = data

}

const isProcessing = ref(false);
const status = ref('');
const errors = ref([]);


async function processMovies() {
  isProcessing.value = true;
  status.value = 'Fetching movies from database...';
  errors.value = [];

  try {
    // Fetch all movies with credits from pbj-movies table
    const { data: movies, error: fetchError } = await supabase
      .from('pbj-movies')
      .select('tmdb_id, credits');

    if (fetchError) throw fetchError;

    if (!movies || movies.length === 0) {
      status.value = 'No movies found in database.';
      return;
    }

    status.value = `Found ${movies.length} movies. Processing actors...`;

    // First, get all existing actors to calculate new appearance counts
    const { data: existingActors } = await supabase
      .from('pbj_actors')
      .select('id, appearances');

    const existingMap = new Map(
      existingActors?.map(a => [a.id, a.appearances]) || []
    );

    // Collect all actors from all movies
    const actorMap = new Map();

    movies.forEach(movie => {
      // Extract cast from credits JSONB
      const credits = movie.credits;

      if (!credits || !credits.cast) {
        errors.value.push(`Movie ${movie.id} has no cast data in credits`);
        return;
      }

      const cast = credits.cast;

      if (!Array.isArray(cast)) {
        errors.value.push(`Movie ${movie.id} has invalid cast data`);
        return;
      }

      cast.forEach(actor => {
        const actorId = actor.id;
        const existingAppearances = existingMap.get(actorId) || 0;

        if (actorMap.has(actorId)) {
          actorMap.get(actorId).newAppearances += 1;
          actorMap.get(actorId).appearances += 1;
        } else {
          actorMap.set(actorId, {
            id: actorId,
            name: actor.name,
            original_name: actor.original_name,
            profile_path: actor.profile_path,
            known_for_department: actor.known_for_department,
            popularity: actor.popularity,
            appearances: existingAppearances + 1,
            newAppearances: 1
          });
        }
      });
    });

    const actors = Array.from(actorMap.values());

    status.value = `Processing ${actors.length} unique actors...`;

    // Use upsert to insert or update in a single operation
    const { error: upsertError } = await supabase
      .from('pbj_actors')
      .upsert(actors, {
        onConflict: 'id',
        ignoreDuplicates: false
      });

    if (upsertError) throw upsertError;

    status.value = `Successfully processed ${actors.length} actors from ${movies.length} movies!`;

  } catch (error) {
    console.error('Error processing movies:', error);
    errors.value.push(`Failed to process movies: ${error.message}`);
    status.value = 'Processing failed. See errors below.';
  } finally {
    isProcessing.value = false;
  }
}

async function topActors() {
  let { data, error } = await supabase.from('pbj_actors').select('id, name, newAppearances').limit(25).order('newAppearances', { ascending: false })
  if (data) topActorsApps.value = data;
}




</script>

<style scoped>
.text-gray-500 {
  color: #6b7280;
  /* Tailwind gray-500 */
}
</style>