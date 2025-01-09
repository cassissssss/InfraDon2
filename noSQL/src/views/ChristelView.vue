<script lang="ts">
import { ref } from 'vue';
import PouchDB from 'pouchdb';
import PouchDBFind from 'pouchdb-find';
PouchDB.plugin(PouchDBFind);
 
// Interfaces existantes
declare interface Comment {
  comment: string;
  author: string;
}
 
declare interface Post {
  _id: string;
  _rev?: string;
  post_name: string;
  post_content: string;
  attributes: {
    creation_date: string;
    author: string;
  },
  comments: Comment[];
}

export default {
  data() {
    return {
      remoteDB: 'http://admin:Cricri1412@localhost:5984/commentaires_data',
      postsData: [] as Post[],
      storage: null as PouchDB.Database | null,
      changes: null as any,
      isEditing: false,
      isAdding: false,
      searchTitle: '',
      addForm: {
        post_name: '',
        post_content: '',
        attributes: {
          author: ''
        },
        comments: [] as Comment[]
      },
      editForm: {
        post_name: '',
        post_content: '',
        _id: '',
        _rev: '',
        attributes: {
          author: ''
        },
        comments: [] as Comment[]
      }
    };
  },
 
  mounted() {
    this.initDatabase();
    this.createTitleIndex();
    this.fetchData();
    this.watchRemoteDatabase();
  },
 
  beforeUnmount() {
    if (this.changes) {
      this.changes.cancel();
    }
  },
 
  methods: {
    async searchByTitle(title: string) {
      const db = ref(this.storage).value
      if (!db) {
        console.error('Database not valid')
        return
      }
 
      try {
        const result = await db.find({
          selector: {
            post_name: { $regex: new RegExp(`.*${title}.*`, 'i') } // Recherche exacte
          }
        })
        console.log('Search results:', result.docs)
        this.postsData = result.docs as Post[] // Met à jour les données affichées
      } catch (error) {
        console.error('Error searching by title:', error)
      }
    },
    resetSearch() {
      this.searchTitle = '' // Réinitialiser le champ de recherche
      this.fetchData() // Recharger tous les documents
    },
    // Initialisation de la base de données
    initDatabase() {
      const localDB = 'local-commentaires';
      const $db = new PouchDB(localDB);
      if ($db) {
        console.log('Connected to collection: ' + $db.name);
      } else {
        console.warn('Something went wrong');
      }
      this.storage = $db;
    },
    async createTitleIndex() {
      const db = ref(this.storage).value
      if (!db) {
        console.error('Database not valid')
        return
      }
      try {
        // Liste des index existants
        const existingIndexes = await db.getIndexes()
        const indexExists = existingIndexes.indexes.some(
          (index) => index.name === 'post_name_index'
        )
 
        if (!indexExists) {
          // Crée l'index uniquement s'il n'existe pas
          await db.createIndex({
            index: {
              fields: ['post_name'],
              name: 'post_name_index' // Nom explicite de l'index
            }
          })
          console.log('Index on "post_name" created successfully')
        } else {
          console.log('Index on "post_name" already exists')
        }
      } catch (error) {
        console.error('Error ensuring index:', error)
      }
    },
 
    // Récupération des données
    fetchData() {
      const db = ref(this.storage).value;
      if (db) {
        db.allDocs({
          include_docs: true,
          attachments: true
        }).then((result: any) => {
          console.log('fetchData success =>', result.rows);
          this.postsData = result.rows.map((row: any) => ({
            _id: row.id,
            _rev: row.doc._rev,
            post_name: row.doc.post_name,
            post_content: row.doc.post_content,
            attributes: {
              creation_date: row.doc.attributes?.creation_date || '',
              author: row.doc.attributes?.author || ''
            },
            comments: row.doc.comments || []
          }));
        }).catch((error: any) => {
          console.log('fetchData error', error);
        });
      }
    },
 
    // Méthodes de synchronisation
    updateLocalDatabase() {
      const db = ref(this.storage).value;
      if (db) {
        db.replicate.from(this.remoteDB)
          .on('complete', () => {
            console.log('on replicate complete');
            this.fetchData();
          })
          .on('error', function (error) {
            console.log('error', error);
          });
      }
    },
 
    updateDistantDatabase() {
      const db = ref(this.storage).value;
      if (db) {
        db.replicate.to(this.remoteDB)
          .on('complete', () => {
            console.log('Synchronisation avec la base distante terminée');
          })
          .on('error', (error) => {
            console.log('Erreur lors de la synchronisation:', error);
          });
      }
    },
    async synchronizeBoth() {
      try {
        const db = ref(this.storage).value;
        if (db) {
          // Synchroniser depuis le serveur d'abord
          await db.replicate.from(this.remoteDB)
            .on('complete', () => {
              console.log('Synchronisation depuis le serveur terminée');
              this.fetchData();
            })
            .on('error', (error) => {
              console.log('Erreur lors de la synchronisation depuis le serveur:', error);
            });

          // Puis synchroniser vers le serveur
          await db.replicate.to(this.remoteDB)
            .on('complete', () => {
              console.log('Synchronisation vers le serveur terminée');
            })
            .on('error', (error) => {
              console.log('Erreur lors de la synchronisation vers le serveur:', error);
            });

          console.log('Synchronisation bidirectionnelle terminée');
        }
      } catch (error) {
        console.error('Erreur lors de la synchronisation bidirectionnelle:', error);
      }
    },
 
    watchRemoteDatabase() {
      const db = ref(this.storage).value;
      if (db) {
        this.changes = db.changes({
          since: 'now',
          live: true,
          include_docs: true
        })
        .on('change', (change) => {
          console.log('Changement détecté:', change);
          this.fetchData();
        })
        .on('error', (error) => {
          console.log('Erreur de watch:', error);
        });
      }
    },
 
    // Méthodes CRUD existantes
    startAdd() {
      this.isAdding = true;
      this.isEditing = false;
    },
 
    addDocument() {
      const newDoc: Post = {
        _id: Date.now().toString(), // Utilisation d'un timestamp comme ID temporaire
        post_name: this.addForm.post_name,
        post_content: this.addForm.post_content,
        attributes: {
          creation_date: new Date().toISOString(),
          author: this.addForm.attributes.author
        },
        comments: this.addForm.comments
      };
 
      const db = ref(this.storage).value;
      if (db) {
        db.put(newDoc).then(() => {
          console.log('Add ok');
          this.fetchData();
          /* this.updateDistantDatabase(); // Synchroniser après l'ajout */
          this.cancelAdd();
        }).catch((error) => {
          console.log('Add ko', error);
        });
      }
    },
 
    cancelAdd() {
      this.isAdding = false;
      this.addForm = {
        post_name: '',
        post_content: '',
        attributes: { author: '' },
        comments: []
      };
    },
 
    addComment() {
      this.addForm.comments.push({ comment: '', author: '' });
    },
 
    deleteComment(index: number) {
      this.addForm.comments.splice(index, 1);
    },
 
    startEdit(post: Post) {
      this.editForm = {
        post_name: post.post_name,
        post_content: post.post_content,
        _id: post._id,
        _rev: post._rev || '',
        attributes: {
          author: post.attributes.author
        },
        comments: post.comments || []
      };
      this.isEditing = true;
      this.isAdding = false;
    },
 
    deleteDocument(post: Post) {
      const db = ref(this.storage).value;
      if (db && post._id && post._rev) {
        console.log('Deleting document:', post);
        db.remove(post._id, post._rev).then(() => {
          console.log('Delete ok');
          this.fetchData();
          this.updateDistantDatabase(); // Synchroniser après la suppression
        }).catch((error) => {
          console.log('Delete ko', error);
        });
      }
    },
 
    saveEdit() {
      const db = ref(this.storage).value;
      if (db && this.editForm._id && this.editForm._rev) {
        const updatedDoc: Post = {
          _id: this.editForm._id,
          _rev: this.editForm._rev,
          post_name: this.editForm.post_name,
          post_content: this.editForm.post_content,
          attributes: {
            creation_date: new Date().toISOString(),
            author: this.editForm.attributes.author
          },
          comments: this.editForm.comments
        };
 
        db.put(updatedDoc).then(() => {
          console.log('Update ok');
          this.fetchData();
          /* this.updateDistantDatabase(); // Synchroniser après la mise à jour */
          this.cancelEdit();
        }).catch((error) => {
          console.log('Update ko', error);
        });
      }
    },
 
    cancelEdit() {
      this.isEditing = false;
      this.editForm = {
        post_name: '',
        post_content: '',
        _id: '',
        _rev: '',
        attributes: { author: '' },
        comments: []
      };
    }
  }
};
</script>
 
<template>
    <div class="search-container">
      <label for="title-search">Rechercher par titre :</label>
      <div class="search-actions">
        <input id="title-search" v-model="searchTitle" placeholder="Entrez un titre" />
        <button @click="searchByTitle(searchTitle)" class="btn search-btn">Rechercher</button>
        <button @click="resetSearch" class="btn cancel">Réinitialiser</button>
      </div>
</div>
  <div class="app-container">
    <header class="header">
      <h1 class="title">Gestion des Posts ({{ postsData.length }})</h1>
      <div class="sync-buttons">
        <button class="btn sync" @click="updateLocalDatabase">
          <span class="icon">↓</span> Synchroniser depuis le serveur
        </button>
        <button class="btn sync" @click="updateDistantDatabase">
          <span class="icon">↑</span> Synchroniser vers le serveur
        </button>
        <button class="btn sync-both" @click="synchronizeBoth">
          <span class="icon">↕</span> Synchronisation bidirectionnelle
        </button>
      </div>
    </header>
 
    <main class="main-content">
      <button class="btn add-btn" @click="startAdd">+ Ajouter un document</button>
 
      <ul class="posts-list">
        <li v-for="post in postsData" :key="post._id" class="post-card">
          <div class="post-header">
            <h2 class="post-title ucfirst">{{ post.post_name }}</h2>
            <div class="post-meta">
              <em v-if="post.attributes.creation_date">
                Créé le: {{ new Date(post.attributes.creation_date).toLocaleDateString() }}
              </em>
            </div>
          </div>
         
          <div class="post-content">{{ post.post_content }}</div>
         
          <div class="post-actions">
            <button class="btn edit" @click="startEdit(post)">Modifier</button>
            <button class="btn delete" @click="deleteDocument(post)">Supprimer</button>
          </div>
        </li>
      </ul>
    </main>
 
    <!-- Formulaire d'ajout -->
    <div v-if="isAdding" class="modal">
      <div class="modal-content">
        <h2>Ajouter un nouveau document</h2>
        <form @submit.prevent="addDocument" class="form">
          <div class="form-group">
            <label for="add_post_name">Nom du post :</label>
            <input type="text" v-model="addForm.post_name" id="add_post_name" required />
          </div>
 
          <div class="form-group">
            <label for="add_post_content">Contenu du post :</label>
            <textarea v-model="addForm.post_content" id="add_post_content" required></textarea>
          </div>
 
          <div class="form-group">
            <label for="add_author">Auteur :</label>
            <input type="text" v-model="addForm.attributes.author" id="add_author" required />
          </div>
 
          <div class="comments-section">
            <h3>Commentaires</h3>
            <div v-for="(comment, index) in addForm.comments" :key="index" class="comment-form">
              <div class="form-group">
                <label :for="'comment' + index">Commentaire :</label>
                <input type="text" v-model="comment.comment" :id="'comment' + index" required />
              </div>
 
              <div class="form-group">
                <label :for="'author' + index">Auteur du commentaire :</label>
                <input type="text" v-model="comment.author" :id="'author' + index" required />
              </div>
 
              <button type="button" class="btn delete small" @click="deleteComment(index)">
                Supprimer le commentaire
              </button>
            </div>
            <button type="button" class="btn add-comment" @click="addComment">
              + Ajouter un commentaire
            </button>
          </div>
 
          <div class="form-actions">
            <button type="submit" class="btn submit">Enregistrer</button>
            <button type="button" class="btn cancel" @click="cancelAdd">Annuler</button>
          </div>
        </form>
      </div>
    </div>
 
    <!-- Formulaire de modification -->
    <div v-if="isEditing" class="modal">
      <div class="modal-content">
        <h2>Modifier le document</h2>
        <form @submit.prevent="saveEdit" class="form">
          <div class="form-group">
            <label for="edit_post_name">Nom du post :</label>
            <input type="text" v-model="editForm.post_name" id="edit_post_name" required />
          </div>
 
          <div class="form-group">
            <label for="edit_post_content">Contenu du post :</label>
            <textarea v-model="editForm.post_content" id="edit_post_content" required></textarea>
          </div>
 
          <div class="form-group">
            <label for="edit_author">Auteur :</label>
            <input type="text" v-model="editForm.attributes.author" id="edit_author" required />
          </div>
 
          <div class="comments-section">
            <h3>Commentaires</h3>
            <div v-for="(comment, index) in editForm.comments" :key="index" class="comment-form">
              <div class="form-group">
                <label :for="'edit_comment' + index">Commentaire :</label>
                <input type="text" v-model="comment.comment" :id="'edit_comment' + index" required />
              </div>
 
              <div class="form-group">
                <label :for="'edit_author_comment' + index">Auteur du commentaire :</label>
                <input type="text" v-model="comment.author" :id="'edit_author_comment' + index" required />
              </div>
            </div>
          </div>
 
          <div class="form-actions">
            <button type="submit" class="btn submit">Enregistrer</button>
            <button type="button" class="btn cancel" @click="cancelEdit">Annuler</button>
          </div>
        </form>
      </div>
    </div>
  </div>
</template>
 
<style scoped>
/* Variables CSS */
:root {
  --primary-color: #3498db;
  --danger-color: #e74c3c;
  --success-color: #2ecc71;
  --warning-color: #f1c40f;
  --text-color: #2c3e50;
  --background-color: #ecf0f1;
  --card-background: #ffffff;
  --border-radius: 8px;
  --spacing: 1rem;
}
 
/* Styles de base */
.app-container {
  max-width: 1200px;
  margin: 0 auto;
  padding: 20px;
  font-family: Arial, sans-serif;
  color: var(--text-color);
}
 
.header {
  margin-bottom: 2rem;
  text-align: center;
}
 
.title {
  color: var(--primary-color);
  margin-bottom: 1rem;
}
 
/* Boutons */
.btn {
  padding: 8px 16px;
  border: none;
  border-radius: var(--border-radius);
  cursor: pointer;
  font-weight: bold;
  transition: all 0.3s ease;
  margin: 0 5px;
}
 
.btn:hover {
  opacity: 0.9;
  transform: translateY(-1px);
}
 
.sync {
  background-color: var(--primary-color);
  color: white;
}

.sync-both {
  background-color: #9b59b6; /* Couleur violette pour distinguer le nouveau bouton */
  color: white;
}
 
.edit {
  background-color: var(--warning-color);
  color: var(--text-color);
}
 
.delete {
  background-color: var(--danger-color);
  color: white;
}
 
.submit {
  background-color: var(--success-color);
  color: white;
}
 
.cancel {
  background-color: #95a5a6;
  color: white;
}
 
.add-btn {
  background-color: var(--success-color);
  color: white;
  font-size: 1.1em;
  margin: 1rem 0;
}
 
/* Liste des posts */
.posts-list {
  list-style: none;
  padding: 0;
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
  gap: 20px;
}
 
.post-card {
  background: var(--card-background);
  border-radius: var(--border-radius);
  padding: var(--spacing);
  box-shadow: 0 2px 5px rgba(0,0,0,0.1);
  transition: transform 0.3s ease;
}
 
.post-card:hover {
  transform: translateY(-3px);
}
 
.post-title {
  margin: 0;
  color: var(--primary-color);
  font-size: 1.2em;
}
 
.post-meta {
  font-size: 0.9em;
  color: #666;
  margin: 5px 0;
}
 
.post-content {
  margin: 10px 0;
  line-height: 1.5;
}
 
.post-actions {
  display: flex;
  justify-content: flex-end;
  gap: 10px;
}
 
/* Modal et formulaires */
.modal {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(0,0,0,0.5);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000;
}
 
.modal-content {
  background: var(--card-background);
  padding: 2rem;
  border-radius: var(--border-radius);
  width: 90%;
  max-width: 600px;
  max-height: 90vh;
  overflow-y: auto;
}
 
.form-group {
  margin-bottom: 1rem;
}
 
.form-group label {
  display: block;
  margin-bottom: 0.5rem;
  font-weight: bold;
}
 
.form-group input,
.form-group textarea {
  width: 100%;
  padding: 8px;
  border: 1px solid #ddd;
  border-radius: var(--border-radius);
  font-size: 1em;
}
 
.form-group textarea {
  min-height: 100px;
  resize: vertical;
}
 
.form-actions {
  display: flex;
  justify-content: flex-end;
  gap: 10px;
  margin-top: 1rem;
}
 
/* Section commentaires */
.comments-section {
  margin: 1rem 0;
  padding: 1rem;
  background: var(--background-color);
  border-radius: var(--border-radius);
}
 
.comment-form {
  margin-bottom: 1rem;
  padding-bottom: 1rem;
  border-bottom: 1px solid #ddd;
}
 
.add-comment {
  background-color: var(--primary-color);
  color: white;
}
 
/* Utilitaires */
.ucfirst {
  text-transform: capitalize;
}
.search-container {
  background: var(--card-background);
  border-radius: var(--border-radius);
  padding: var(--spacing);
  box-shadow: 0 2px 5px rgba(0,0,0,0.1);
  margin-bottom: 2rem;
}

.search-container label {
  display: block;
  margin-bottom: 0.5rem;
  font-weight: bold;
  color: var(--text-color);
}

.search-actions {
  display: flex;
  gap: 10px;
  align-items: center;
}

#title-search {
  flex: 1;
  padding: 8px;
  border: 1px solid #ddd;
  border-radius: var(--border-radius);
  font-size: 1em;
  color: white;
  background-color: var(--text-color);
}

#title-search::placeholder {
  color: rgba(255, 255, 255, 0.7);
}

#title-search:focus {
  outline: none;
  border-color: var(--primary-color);
  box-shadow: 0 0 0 2px rgba(52, 152, 219, 0.1);
}

.search-btn {
  padding: 8px 16px;
  border: none;
  border-radius: var(--border-radius);
  cursor: pointer;
  font-weight: bold;
  transition: all 0.3s ease;
  background-color: var(--primary-color);
  color: white;
}

.search-btn:hover {
  opacity: 0.9;
  transform: translateY(-1px);
}

@media (max-width: 768px) {
  .search-container {
    padding: var(--spacing);
  }
  
  .search-actions {
    flex-direction: column;
    gap: 8px;
  }
  
  #title-search {
    width: 100%;
  }
  
  .search-btn, 
  .cancel {
    width: 100%;
    padding: 6px 12px;
    font-size: 0.9em;
  }
}
/* Media queries pour la responsivité */
@media (max-width: 768px) {
  .posts-list {
    grid-template-columns: 1fr;
  }
 
  .modal-content {
    width: 95%;
    padding: 1rem;
  }
 
  .btn {
    padding: 6px 12px;
    font-size: 0.9em;
  }
}
</style>