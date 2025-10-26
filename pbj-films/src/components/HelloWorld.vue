<template>
  <v-container class="fill-height" max-width="900">
    <div>
      <v-img class="mb-4" height="150" src="@/assets/logo.png" />

      <div class="mb-8 text-center">
        <div class="text-body-2 font-weight-light mb-n1">Welcome to</div>
        <h1 class="text-h2 font-weight-bold">PBJ Films</h1>
      </div>

      <v-row>
        <v-col v-for="movie in movies" :key="movie.tmdb_id" cols="6">
          <v-card :title="movie.title" :subtitle="movie.date_watched" rounded="lg" class="mx-auto">
            <v-card-text>
              <ul>
                <li v-for="person in movie.credits.cast" :key="person.id">
                  {{ person.name }} <span class="text-gray-500">as</span> {{ person.character }}
                </li>
              </ul>
            </v-card-text>

          </v-card>
        </v-col>

      </v-row>

    </div>
  </v-container>
</template>

<script setup>
const links = [
  {
    href: 'https://vuetifyjs.com/',
    icon: 'mdi-text-box-outline',
    subtitle: 'Learn about all things Vuetify in our documentation.',
    title: 'Docs',
  },
  {
    href: 'https://vuetifyjs.com/introduction/why-vuetify/#feature-guides',
    icon: 'mdi-star-circle-outline',
    subtitle: 'Explore available framework Features.',
    title: 'Features',
  },
  {
    href: 'https://vuetifyjs.com/components/all',
    icon: 'mdi-widgets-outline',
    subtitle: 'Discover components in the API Explorer.',
    title: 'Components',
  },
]

import { ref, onMounted } from 'vue'
import { supabase } from '../lib/supabaseClient'
const movies = ref([])

// async function getMovies() {
//   const { data } = await supabase.from('pbj-movies').select()
//   movies.value = data
// }

onMounted(() => {
  loadMovies()
})


const loadMovies = async () => {

  let { data, error
  } = await supabase
    .from('pbj-movies')
    .select('*')

  console.log('data: ', data)
  if (data) movies.value = data

}

</script>

<style scoped>
.text-gray-500 {
  color: #6b7280; /* Tailwind gray-500 */
}
</style>