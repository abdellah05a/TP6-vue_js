<template>
  <div v-if="student">
    <h1>{{ student.firstName }} {{ student.lastName }}</h1>

    <p><label>Filière : <input v-model.trim="student.filiere" type="text" /></label></p>
    <p><label>Année : <input v-model.number="student.year" type="number" min="1" /></label></p>
    <p><label>Email : <input v-model.trim="student.email" type="email" /></label></p>
  </div>
  <div v-else>
    <p>Étudiant introuvable.</p>
  </div>

  <button @click="$router.back()">Retour</button>
</template>

<script>
import { store } from '../store/students.js'

export default {
  name: 'StudentDetailView',
  computed: {
    student() {
      const id = Number(this.$route.params.id)
      return store.students.find((student) => student.id === id)
    }
  }
}
</script>
