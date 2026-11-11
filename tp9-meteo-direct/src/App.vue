<!--
  TP9 -- Meteo en direct.
  Le gabarit (template) est deja cable sur les donnees : il n'y a rien a
  modifier dans la partie template. Tout le travail est dans les methodes
  et la computed de la partie script, marquees TODO 1 a 6.
-->
<template>
  <div class="app">
    <header class="topbar">
      <h1>Météo en direct</h1>
      <button class="unit-toggle" @click="basculerUnite" v-if="ville">
        Afficher en {{ unite === 'metric' ? '°F' : '°C' }}
      </button>
    </header>

    <div class="search">
      <input
        v-model="query"
        @input="onInputRecherche"
        type="text"
        placeholder="Rechercher une ville... (ex: Safi, Paris, Tokyo)"
      />
      <p class="hint" v-if="chargementSuggestions">Recherche...</p>
      <p class="msg-error" v-if="erreurSuggestions">{{ erreurSuggestions }}</p>

      <ul class="suggestions" v-if="suggestions.length">
        <li v-for="s in suggestions" :key="s.lat + '-' + s.lon" @click="choisirVille(s)">
          {{ s.name }}<span v-if="s.state">, {{ s.state }}</span> ({{ s.country }})
        </li>
      </ul>
    </div>

    <div class="historique" v-if="historique.length">
      <span>Récent :</span>
      <button v-for="h in historique" :key="h.lat + '-' + h.lon" @click="choisirVille(h)">
        {{ h.name }}
      </button>
    </div>

    <div class="meteo" v-if="ville">
      <p class="loading" v-if="chargementMeteo">Chargement de la météo...</p>
      <p class="msg-error" v-else-if="erreurMeteo">{{ erreurMeteo }}</p>

      <div v-else-if="meteoActuelle" class="courante">
        <h2>{{ ville.name }}, {{ ville.country }}</h2>
        <div class="temp-actuelle">
          <img :src="iconeUrl(meteoActuelle.weather[0].icon)" :alt="meteoActuelle.weather[0].description" />
          <span class="valeur">{{ Math.round(meteoActuelle.main.temp) }}°</span>
          <span class="description">{{ meteoActuelle.weather[0].description }}</span>
        </div>
        <p class="details">
          Ressenti {{ Math.round(meteoActuelle.main.feels_like) }}° · Humidité {{ meteoActuelle.main.humidity }}%
        </p>
      </div>

      <div class="previsions" v-if="previsionsParJour.length">
        <div class="jour" v-for="j in previsionsParJour" :key="j.date">
          <span class="date">{{ j.date }}</span>
          <img :src="iconeUrl(j.icone)" :alt="j.description" />
          <span class="temps">{{ Math.round(j.min) }}° / {{ Math.round(j.max) }}°</span>
        </div>
      </div>
    </div>

    <p class="empty" v-if="!ville">Cherchez une ville pour voir sa météo.</p>
  </div>
</template>

<script>
// La cle est lue depuis .env (jamais committee -- voir .env.example et le sujet).
const API_KEY = import.meta.env.VITE_OWM_API_KEY
const GEO_URL = 'https://api.openweathermap.org/geo/1.0/direct'
const WEATHER_URL = 'https://api.openweathermap.org/data/2.5/weather'
const FORECAST_URL = 'https://api.openweathermap.org/data/2.5/forecast'

export default {
  name: 'App',
  data() {
    return {
      query: '',
      debounceTimer: null,
      suggestions: [],
      chargementSuggestions: false,
      erreurSuggestions: null,

      ville: null, // { name, country, state, lat, lon }
      historique: [],

      meteoActuelle: null,
      previsions: [], // tableau brut renvoye par /forecast (tranches de 3h)
      chargementMeteo: false,
      erreurMeteo: null,

      unite: 'metric' // 'metric' (°C) | 'imperial' (°F)
    }
  },
  computed: {
    // TODO 6 : regroupez this.previsions (tranches de 3h, sur 5 jours) en une
    // prevision par jour. Chaque element de this.previsions a la forme :
    //   { dt_txt: "2024-05-12 15:00:00", main: { temp_min, temp_max }, weather: [{ icon, description }] }
    // Indice : la partie de dt_txt avant l'espace ("2024-05-12") est une bonne
    // cle de regroupement. Pour chaque jour, calculez le min et le max des
    // temperatures rencontrees, et gardez une icone/description representative
    // (par exemple celle de la tranche la plus proche de midi).
    // Renvoyez un tableau trie par date : [{ date, min, max, icone, description }, ...]
    previsionsParJour() {
      const jours = new Map()

      this.previsions.forEach((prevision) => {
        const date = prevision.dt_txt.split(' ')[0]
        const heure = Number(prevision.dt_txt.slice(11, 13))
        const distanceDeMidi = Math.abs(heure - 12)

        if (!jours.has(date)) {
          jours.set(date, {
            date,
            min: prevision.main.temp_min,
            max: prevision.main.temp_max,
            icone: prevision.weather[0].icon,
            description: prevision.weather[0].description,
            distanceDeMidi
          })
          return
        }

        const jour = jours.get(date)
        jour.min = Math.min(jour.min, prevision.main.temp_min)
        jour.max = Math.max(jour.max, prevision.main.temp_max)

        if (distanceDeMidi < jour.distanceDeMidi) {
          jour.icone = prevision.weather[0].icon
          jour.description = prevision.weather[0].description
          jour.distanceDeMidi = distanceDeMidi
        }
      })

      return [...jours.values()]
        .sort((a, b) => a.date.localeCompare(b.date))
        .map(({ distanceDeMidi, ...jour }) => jour)
    }
  },
  methods: {
    iconeUrl(code) {
      return `https://openweathermap.org/img/wn/${code}@2x.png`
    },

    // TODO 1 : declenchee a chaque frappe dans le champ de recherche.
    // Elle doit annuler le minuteur precedent (clearTimeout(this.debounceTimer))
    // puis en programmer un nouveau (setTimeout) qui appelle rechercherVilles()
    // apres ~400ms. Si this.query est vide, videz suggestions et ne programmez rien.
    onInputRecherche() {
      clearTimeout(this.debounceTimer)
      this.erreurSuggestions = null

      if (!this.query.trim()) {
        this.suggestions = []
        this.chargementSuggestions = false
        return
      }

      this.debounceTimer = setTimeout(() => {
        this.rechercherVilles()
      }, 400)
    },

    // TODO 2 : appelle l'API de geocodage pour retrouver les villes correspondant
    // a this.query, et remplit this.suggestions.
    // URL : `${GEO_URL}?q=${encodeURIComponent(this.query)}&limit=5&appid=${API_KEY}`
    // A gerer avec des messages differents dans erreurSuggestions :
    //   - reponse.status === 401 -> cle API invalide ou absente
    //   - reponse.status === 429 -> quota depasse, reessayer plus tard
    //   - reponse.ok mais tableau vide -> aucune ville trouvee pour cette recherche
    // Pensez a chargementSuggestions (true avant l'appel, false dans un finally).
    async rechercherVilles() {
      const recherche = this.query.trim()

      if (!recherche) {
        this.suggestions = []
        return
      }

      this.chargementSuggestions = true
      this.erreurSuggestions = null

      try {
        const response = await fetch(
          `${GEO_URL}?q=${encodeURIComponent(recherche)}&limit=5&appid=${API_KEY}`
        )

        if (response.status === 401) {
          this.erreurSuggestions = 'Clé API invalide ou absente. Vérifiez le fichier .env.'
          return
        }

        if (response.status === 429) {
          this.erreurSuggestions = 'Quota dépassé. Réessayez plus tard.'
          return
        }

        if (!response.ok) {
          throw new Error(`Erreur HTTP ${response.status}`)
        }

        const villes = await response.json()

        // Une recherche plus récente a pu être lancée pendant cette requête.
        if (this.query.trim() !== recherche) return

        this.suggestions = villes
        if (!villes.length) {
          this.erreurSuggestions = 'Aucune ville trouvée pour cette recherche.'
        }
      } catch (error) {
        this.erreurSuggestions = `Impossible de rechercher les villes : ${error.message}`
      } finally {
        if (this.query.trim() === recherche) {
          this.chargementSuggestions = false
        }
      }
    },

    // TODO 3 : appelee au clic sur une suggestion OU sur un element de l'historique.
    // Doit : vider suggestions et query, memoriser la ville dans this.ville,
    // ajouter cette ville en tete de this.historique (pas de doublon sur le
    // couple lat/lon, 5 elements maximum), puis appeler chargerMeteo().
    choisirVille(ville) {
      clearTimeout(this.debounceTimer)
      this.suggestions = []
      this.query = ''
      this.erreurSuggestions = null
      this.ville = ville
      this.historique = [
        ville,
        ...this.historique.filter((element) =>
          element.lat !== ville.lat || element.lon !== ville.lon
        )
      ].slice(0, 5)
      this.chargerMeteo()
    },

    // TODO 4 : charge la meteo actuelle ET les previsions pour this.ville, en
    // lancant les deux requetes EN PARALLELE avec Promise.all plutot que l'une
    // apres l'autre (elles ne dependent pas l'une de l'autre).
    //   - meteo actuelle : `${WEATHER_URL}?lat=${lat}&lon=${lon}&appid=${API_KEY}&units=${this.unite}&lang=fr`
    //   - previsions      : `${FORECAST_URL}?lat=${lat}&lon=${lon}&appid=${API_KEY}&units=${this.unite}&lang=fr`
    // Remplissez this.meteoActuelle (objet) et this.previsions (donnees.list,
    // un tableau -- la computed previsionsParJour se charge de l'agregation).
    // Gerez chargementMeteo/erreurMeteo comme dans les TP precedents.
    async chargerMeteo() {
      if (!this.ville) return

      this.chargementMeteo = true
      this.erreurMeteo = null
      this.meteoActuelle = null
      this.previsions = []

      try {
        if (!API_KEY || API_KEY === 'votre_cle_ici') {
          throw new Error('Clé API absente. Créez un fichier .env à partir de .env.example.')
        }

        const { lat, lon } = this.ville
        const params = `lat=${lat}&lon=${lon}&appid=${API_KEY}&units=${this.unite}&lang=fr`
        const [meteoResponse, previsionsResponse] = await Promise.all([
          fetch(`${WEATHER_URL}?${params}`),
          fetch(`${FORECAST_URL}?${params}`)
        ])

        const reponseEnErreur = [meteoResponse, previsionsResponse].find(
          (response) => !response.ok
        )
        if (reponseEnErreur) {
          if (reponseEnErreur.status === 401) {
            throw new Error('Clé API invalide ou absente. Vérifiez le fichier .env.')
          }
          if (reponseEnErreur.status === 429) {
            throw new Error('Quota dépassé. Réessayez plus tard.')
          }
          throw new Error(`Erreur HTTP ${reponseEnErreur.status}`)
        }

        const [meteoActuelle, donneesPrevisions] = await Promise.all([
          meteoResponse.json(),
          previsionsResponse.json()
        ])
        this.meteoActuelle = meteoActuelle
        this.previsions = donneesPrevisions.list
      } catch (error) {
        this.erreurMeteo = `Impossible de charger la météo : ${error.message}`
      } finally {
        this.chargementMeteo = false
      }
    },

    // TODO 5 : bascule this.unite entre 'metric' et 'imperial', puis recharge
    // la meteo de la ville courante (les temperatures dependent du parametre
    // units= dans l'URL, donc changer l'unite veut dire refaire l'appel).
    basculerUnite() {
      this.unite = this.unite === 'metric' ? 'imperial' : 'metric'
      if (this.ville) {
        this.chargerMeteo()
      }
    }
  }
}
</script>
