<template>
  <h1>Explorateur de pays</h1>

  <p class="loading" v-show="loading">Chargement des pays...</p>
  <p class="error" v-if="error">{{ error }}</p>

  <div class="toolbar">
    <input v-model="recherche" placeholder="Rechercher un pays..." />
  </div>

  <div class="grid">
    <CountryCard v-for="country in paysAffiches" :key="country.cca3" :country="country" />
  </div>
</template>

<script>
import CountryCard from '../components/CountryCard.vue'

const API_URL = 'https://pw-d20261017a29.plainwork-apps.workers.dev/v3.1/all?fields=name,capital,region,population,flags,cca3'

export default {
  name: 'CountryListView',
  components: { CountryCard },
  data() {
    return {
      countries: [],
      loading: true,
      error: null,
      recherche: ''
    }
  },
  computed: {
    paysAffiches() {
      const recherche = this.recherche.trim().toLocaleLowerCase()

      if (!recherche) {
        return this.countries
      }

      return this.countries.filter((country) =>
        country.name.common.toLocaleLowerCase().includes(recherche)
      )
    }
  },
  methods: {
    async chargerPays() {
      this.loading = true
      this.error = null

      try {
        const response = await fetch(API_URL)

        if (!response.ok) {
          throw new Error(`Erreur HTTP ${response.status}`)
        }

        const countries = await response.json()
        this.countries = countries.sort((a, b) =>
          a.name.common.localeCompare(b.name.common, 'fr')
        )
      } catch (error) {
        this.error = `Impossible de charger les pays : ${error.message}`
      } finally {
        this.loading = false
      }
    }
  },
  mounted() {
    this.chargerPays()
  }
}
</script>
