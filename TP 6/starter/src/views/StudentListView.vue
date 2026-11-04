<template>
  <h1>Annuaire des étudiants</h1>

  <form @submit.prevent="ajouterEtudiant">
    <input v-model.trim="newStudent.firstName" type="text" placeholder="Prénom *" aria-label="Prénom" />
    <input v-model.trim="newStudent.lastName" type="text" placeholder="Nom *" aria-label="Nom" />
    <input v-model.trim="newStudent.filiere" type="text" placeholder="Filière" aria-label="Filière" />
    <input v-model.number="newStudent.year" type="number" min="1" placeholder="Année" aria-label="Année" />
    <input v-model.trim="newStudent.email" type="email" placeholder="Email" aria-label="Email" />
    <button type="submit">Ajouter</button>
    <p v-if="error" class="error" role="alert">{{ error }}</p>
  </form>

  <div class="toolbar">
    <input v-model.trim="recherche" type="search" placeholder="Rechercher un étudiant" aria-label="Rechercher" />
    <label>
      Trier par
      <select v-model="tri">
        <option value="lastName">Nom</option>
        <option value="filiere">Filière</option>
      </select>
    </label>
  </div>

  <StudentCard v-for="student in etudiantsAffiches" :key="student.id" :student="student" />
  <p v-if="etudiantsAffiches.length === 0">Aucun étudiant ne correspond à votre recherche.</p>
</template>

<script>
import StudentCard from '../components/StudentCard.vue'
import { store } from '../store/students.js'

export default {
  name: 'StudentListView',
  components: { StudentCard },
  data() {
    return {
      store,
      recherche: '',
      tri: 'lastName',
      newStudent: { firstName: '', lastName: '', filiere: '', year: 1, email: '' },
      error: ''
    }
  },
  computed: {
    etudiantsAffiches() {
      const recherche = this.recherche.toLocaleLowerCase()
      return [...this.store.students]
        .filter((student) =>
          `${student.firstName} ${student.lastName}`.toLocaleLowerCase().includes(recherche)
        )
        .sort((a, b) =>
          a[this.tri].localeCompare(b[this.tri], 'fr', { sensitivity: 'base' })
        )
    }
  },
  methods: {
    ajouterEtudiant() {
      if (!this.newStudent.firstName || !this.newStudent.lastName) {
        this.error = 'Le prénom et le nom sont obligatoires.'
        return
      }

      const newId = Math.max(0, ...this.store.students.map((student) => student.id)) + 1
      this.store.students.push({ ...this.newStudent, id: newId })
      this.newStudent = { firstName: '', lastName: '', filiere: '', year: 1, email: '' }
      this.error = ''
    }
  }
}
</script>
