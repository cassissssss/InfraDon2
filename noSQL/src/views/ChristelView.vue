<script lang="ts">
//import PouchDB from 'pouchdb';
export default {
  data() {
    return {
      datas: [],
      databaseReference: null,
    };
  },
  methods: {
    // Initialisation de la base de données
    initDatabase() {
      const db = new PouchDB('/commentaires_data');
      this.databaseReference = db;

      // Récupération des documents après l'initialisation
      this.fetchData();
    },

    // Récupération des documents depuis la base de données
    fetchData() {
      if (!this.databaseReference) {
        console.error("Base de données non initialisée !");
        return;
      }

      // Utilisation de allDocs pour récupérer tous les documents
      this.databaseReference.allDocs({ include_docs: true }) // include_docs: true permet de récupérer le contenu complet des documents
        .then((result: AllDocsResponse<{}>) => {
          // Les documents sont dans result.rows (tableau de documents)
          this.datas = result.rows.map(row => row.doc); // Stocker les docs dans datas
          console.log("Documents récupérés :", this.datas);
        })
        .catch((error: any) => {
          console.error("Erreur lors de la récupération des documents :", error);
        });
    },
  },
  
  mounted() {
    this.initDatabase()
  }
}
</script>
<template>
  <h1>InfraDon2</h1>
  <!----<p>Counter: {{ datas }}</p><
  <button type="button" role="button" @click="inc">
    +1
  </button>---->
</template>