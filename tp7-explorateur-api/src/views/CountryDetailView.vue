<template>
  <p class="loading" v-show="loading">Chargement...</p>
  <p class="error" v-if="error">{{ error }}</p>

  <div v-if="country">
    <img :src="country.flags.svg" :alt="`Drapeau ${country.name.common}`" style="width: 200px;" />
    <h1>{{ country.name.common }}</h1>
    <p>Nom officiel : {{ country.name.official }}</p>
    <p>Capitale : {{ country.capital ? country.capital[0] : '—' }}</p>
    <p>Région : {{ country.region }}</p>
    <p>Population : {{ country.population.toLocaleString() }}</p>
  </div>

  <button @click="$router.back()">Retour</button>
</template>

<script>
const FIELDS = 'name,capital,region,population,flags,cca3'
const API_BASE_URL = 'https://pw-d20261017a29.plainwork-apps.workers.dev/v3.1'

export default {
  name: 'CountryDetailView',
  data() {
    return {
      country: null,
      loading: true,
      error: null
    }
  },
  methods: {
    async chargerPays() {
      this.loading = true
      this.error = null
      this.country = null

      try {
        const code = encodeURIComponent(this.$route.params.code)
        const response = await fetch(
          `${API_BASE_URL}/alpha/${code}?fields=${FIELDS}`
        )

        if (!response.ok) {
          throw new Error(`Erreur HTTP ${response.status}`)
        }

        const data = await response.json()
        this.country = Array.isArray(data) ? data[0] : data

        if (!this.country) {
          throw new Error('Pays introuvable')
        }
      } catch (error) {
        this.error = `Impossible de charger ce pays : ${error.message}`
      } finally {
        this.loading = false
      }
    }
  },
  watch: {
    '$route.params.code': 'chargerPays'
  },
  mounted() {
    this.chargerPays()
  }
}
</script>
