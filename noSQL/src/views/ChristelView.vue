<script lang="ts">
import { ref } from 'vue';
import PouchDB from 'pouchdb';
import PouchDBFind from 'pouchdb-find';
PouchDB.plugin(PouchDBFind);

// Interfaces
declare interface Comment {
  comment: string;
  author: string;
}

declare interface Post {
  _id: string;
  _rev?: string;
  post_name: string;
  post_content: string;
  image_data?: string;
  attributes: {
    creation_date: string;
    author: string;
  },
  comments: Comment[];
}

// Fonction utilitaire pour convertir les fichiers en base64
const convertToBase64 = (file: File): Promise<string> => {
  return new Promise((resolve, reject) => {
    const reader = new FileReader();
    reader.readAsDataURL(file);
    reader.onload = () => resolve(reader.result as string);
    reader.onerror = error => reject(error);
  });
};

// Fonction utilitaire pour générer du contenu aléatoire
const generateRandomContent = () => {
  const titles = ['Mon super article', 'Une journée incroyable', 'Découverte passionnante', 
                 'Actualités du jour', 'Revue technologique', 'Le saviez-vous ?'];
  const contents = ['Voici un contenu très intéressant...', 'Aujourd\'hui, nous allons parler de...', 
                   'Dans cet article, découvrez...', 'Une nouvelle approche pour...', 
                   'Les dernières avancées dans...'];
  const authors = ['Jean Dupont', 'Marie Martin', 'Pierre Durant', 'Sophie Bernard', 'Lucas Petit'];
  
  return {
    title: titles[Math.floor(Math.random() * titles.length)],
    content: contents[Math.floor(Math.random() * contents.length)],
    author: authors[Math.floor(Math.random() * authors.length)]
  };
};

export default {
  data() {
    return {
      remoteDB: 'http://admin:Cricri1412@localhost:5984/commentaires_data',
      postsData: [] as Post[],
      storage: null as PouchDB.Database | null,
      changes: null as any,
      isEditing: false,
      isAdding: false,
      isGenerating: false,
      searchTitle: '',
      generationCount: 1,
      addForm: {
        post_name: '',
        post_content: '',
        image_data: '',
        image_file: null as File | null,
        attributes: {
          author: ''
        },
        comments: [] as Comment[]
      },
      editForm: {
        post_name: '',
        post_content: '',
        image_data: '',
        image_file: null as File | null,
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
    // Gestion des images
    async handleImageUpload(event: Event, form: 'add' | 'edit') {
      const file = (event.target as HTMLInputElement).files?.[0];
      if (!file) return;

      try {
        const base64Data = await convertToBase64(file);
        if (form === 'add') {
          this.addForm.image_data = base64Data;
          this.addForm.image_file = file;
        } else {
          this.editForm.image_data = base64Data;
          this.editForm.image_file = file;
        }
      } catch (error) {
        console.error('Erreur lors de la conversion de l\'image:', error);
      }
    },

    // Recherche
    async searchByTitle(title: string) {
      const db = ref(this.storage).value;
      if (!db) {
        console.error('Database not valid');
        return;
      }

      try {
        const result = await db.find({
          selector: {
            post_name: { $regex: new RegExp(`.*${title}.*`, 'i') }
          }
        });
        this.postsData = result.docs as Post[];
      } catch (error) {
        console.error('Error searching by title:', error);
      }
    },

    resetSearch() {
      this.searchTitle = '';
      this.fetchData();
    },

    // Initialisation et gestion de la base de données
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
      const db = ref(this.storage).value;
      if (!db) {
        console.error('Database not valid');
        return;
      }

      try {
        const existingIndexes = await db.getIndexes();
        const indexExists = existingIndexes.indexes.some(
          (index) => index.name === 'post_name_index'
        );

        if (!indexExists) {
          await db.createIndex({
            index: {
              fields: ['post_name'],
              name: 'post_name_index'
            }
          });
          console.log('Index on "post_name" created successfully');
        }
      } catch (error) {
        console.error('Error ensuring index:', error);
      }
    },

    // Génération de posts
    async generatePosts() {
      const db = ref(this.storage).value;
      if (!db) return;

      const count = parseInt(this.generationCount.toString());
      if (isNaN(count) || count <= 0) {
        console.error('Nombre invalide de posts à générer');
        return;
      }

      try {
        for (let i = 0; i < count; i++) {
          const randomContent = generateRandomContent();
          const newDoc: Post = {
            _id: `generated_${Date.now()}_${i}`,
            post_name: randomContent.title,
            post_content: randomContent.content,
            attributes: {
              creation_date: new Date().toISOString(),
              author: randomContent.author
            },
            comments: []
          };

          await db.put(newDoc);
        }
        
        console.log(`${count} posts générés avec succès`);
        this.fetchData();
        this.isGenerating = false;
      } catch (error) {
        console.error('Erreur lors de la génération des posts:', error);
      }
    },

    // CRUD Operations
    async fetchData() {
      const db = ref(this.storage).value;
      if (db) {
        try {
          const result = await db.allDocs({
            include_docs: true,
            attachments: true
          });
          this.postsData = result.rows.map((row: any) => ({
            _id: row.id,
            _rev: row.doc._rev,
            post_name: row.doc.post_name,
            post_content: row.doc.post_content,
            image_data: row.doc.image_data,
            attributes: {
              creation_date: row.doc.attributes?.creation_date || '',
              author: row.doc.attributes?.author || ''
            },
            comments: row.doc.comments || []
          }));
        } catch (error) {
          console.error('fetchData error', error);
        }
      }
    },

    async addDocument() {
      const newDoc: Post = {
        _id: Date.now().toString(),
        post_name: this.addForm.post_name,
        post_content: this.addForm.post_content,
        image_data: this.addForm.image_data || undefined,
        attributes: {
          creation_date: new Date().toISOString(),
          author: this.addForm.attributes.author
        },
        comments: this.addForm.comments
      };

      const db = ref(this.storage).value;
      if (db) {
        try {
          await db.put(newDoc);
          console.log('Add ok');
          this.fetchData();
          this.cancelAdd();
        } catch (error) {
          console.log('Add ko', error);
        }
      }
    },

    startEdit(post: Post) {
      this.editForm = {
        post_name: post.post_name,
        post_content: post.post_content,
        image_data: post.image_data || '',
        image_file: null,
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

    async saveEdit() {
      const db = ref(this.storage).value;
      if (db && this.editForm._id && this.editForm._rev) {
        const updatedDoc: Post = {
          _id: this.editForm._id,
          _rev: this.editForm._rev,
          post_name: this.editForm.post_name,
          post_content: this.editForm.post_content,
          image_data: this.editForm.image_data || undefined,
          attributes: {
            creation_date: new Date().toISOString(),
            author: this.editForm.attributes.author
          },
          comments: this.editForm.comments
        };

        try {
          await db.put(updatedDoc);
          console.log('Update ok');
          this.fetchData();
          this.cancelEdit();
        } catch (error) {
          console.log('Update ko', error);
        }
      }
    },

    // Gestion des formulaires
    startAdd() {
      this.isAdding = true;
      this.isEditing = false;
    },

    cancelAdd() {
      this.isAdding = false;
      this.addForm = {
        post_name: '',
        post_content: '',
        image_data: '',
        image_file: null,
        attributes: { author: '' },
        comments: []
      };
    },

    cancelEdit() {
      this.isEditing = false;
      this.editForm = {
        post_name: '',
        post_content: '',
        image_data: '',
        image_file: null,
        _id: '',
        _rev: '',
        attributes: { author: '' },
        comments: []
      };
    },

    // Gestion des commentaires
    addComment() {
      this.addForm.comments.push({ comment: '', author: '' });
    },

    deleteComment(index: number) {
      this.addForm.comments.splice(index, 1);
    },

    // Suppression de document
    async deleteDocument(post: Post) {
      const db = ref(this.storage).value;
      if (db && post._id && post._rev) {
        try {
          await db.remove(post._id, post._rev);
          console.log('Delete ok');
          this.fetchData();
        } catch (error) {
          console.log('Delete ko', error);
        }
      }
    },

    // Synchronisation
    async synchronizeBoth() {
      try {
        const db = ref(this.storage).value;
        if (db) {
          await db.replicate.from(this.remoteDB)
            .on('complete', () => {
              console.log('Synchronisation depuis le serveur terminée');
              this.fetchData();
            });

          await db.replicate.to(this.remoteDB)
            .on('complete', () => {
              console.log('Synchronisation vers le serveur terminée');
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
        .on('change', () => {
          this.fetchData();
        })
        .on('error', (error) => {
          console.log('Erreur de watch:', error);
        });
      }
    }
  }
};
</script>

<template>
  <div class="app-container">
    <!-- Barre de recherche -->
    <div class="search-container">
      <label for="title-search">Rechercher par titre :</label>
      <div class="search-actions">
        <input id="title-search" v-model="searchTitle" placeholder="Entrez un titre" />
        <button @click="searchByTitle(searchTitle)" class="btn search-btn">Rechercher</button>
        <button @click="resetSearch" class="btn cancel">Réinitialiser</button>
      </div>
    </div>

    <!-- En-tête -->
    <header class="header">
      <h1 class="title">Gestion des Posts ({{ postsData.length }})</h1>
      <div class="action-buttons">
        <div class="sync-buttons">
          <button class="btn sync-both" @click="synchronizeBoth">
            <span class="icon">↕</span> Synchronisation bidirectionnelle
          </button>
        </div>
        <button class="btn generate" @click="isGenerating = true">
          Générer des posts
        </button>
      </div>
    </header>

    <!-- Contenu principal -->
    <main class="main-content">
      <button class="btn add-btn" @click="startAdd">+ Ajouter un document</button>

      <!-- Liste des posts -->
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

          <!-- Affichage de l'image -->
          <img 
            v-if="post.image_data" 
            :src="post.image_data" 
            :alt="post.post_name"
            class="post-image" 
          />
          
          <div class="post-content">{{ post.post_content }}</div>
          
          <div class="post-actions">
            <button class="btn edit" @click="startEdit(post)">Modifier</button>
            <button class="btn delete" @click="deleteDocument(post)">Supprimer</button>
          </div>
        </li>
      </ul>
    </main>

    <!-- Modal de génération -->
    <div v-if="isGenerating" class="modal">
      <div class="modal-content">

        <h2>Générer des posts</h2>
        <div class="form-group">
          <label for="generation-count">Nombre de posts à générer :</label>
          <input
            type="number"
            id="generation-count"
            v-model="generationCount"
            min="1"
            required
          />
        </div>
        <div class="form-actions">
          <button @click="generatePosts" class="btn submit">Générer</button>
          <button @click="isGenerating = false" class="btn cancel">Annuler</button>
        </div>
      </div>
    </div>

    <!-- Modal d'ajout -->
    <div v-if="isAdding" class="modal">
      <div class="modal-content">
        <h2>Ajouter un nouveau document</h2>
        <form @submit.prevent="addDocument" class="form">
          <div class="form-group">
            <label for="add_post_name">Nom du post :</label>
            <input type="text" v-model="addForm.post_name" id="add_post_name" required />
          </div>

          <div class="form-group">
            <label for="add_image">Image :</label>
            <input 
              type="file" 
              id="add_image" 
              @change="handleImageUpload($event, 'add')" 
              accept="image/*"
              class="image-input"
            />
            <img 
              v-if="addForm.image_data" 
              :src="addForm.image_data" 
              alt="Aperçu" 
              class="image-preview"
            />
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

    <!-- Modal de modification -->
    <div v-if="isEditing" class="modal">
      <div class="modal-content">
        <h2>Modifier le document</h2>
        <form @submit.prevent="saveEdit" class="form">
          <div class="form-group">
            <label for="edit_post_name">Nom du post :</label>
            <input type="text" v-model="editForm.post_name" id="edit_post_name" required />
          </div>

          <div class="form-group">
            <label for="edit_image">Image :</label>
            <input 
              type="file" 
              id="edit_image" 
              @change="handleImageUpload($event, 'edit')" 
              accept="image/*"
              class="image-input"
            />
            <img 
              v-if="editForm.image_data" 
              :src="editForm.image_data" 
              alt="Aperçu" 
              class="image-preview"
            />
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

.action-buttons {
  display: flex;
  justify-content: center;
  gap: 1rem;
  margin-bottom: 1rem;
}

/* Styles spécifiques à l'upload d'images */
.image-input {
  width: 100%;
  padding: 8px;
  margin-bottom: 10px;
}

.image-preview {
  max-width: 100%;
  max-height: 200px;
  object-fit: contain;
  margin: 10px 0;
  border-radius: var(--border-radius);
}

.post-image {
  width: 100%;
  max-height: 300px;
  object-fit: cover;
  border-radius: var(--border-radius);
  margin: 10px 0;
}

/* Boutons */
.btn {
  padding: 8px 16px;
  border: none;
  border-radius: var(--border-radius);
  cursor: pointer;
  font-weight: bold;
  transition: all 0.3s ease;
}

.btn:hover {
  opacity: 0.9;
  transform: translateY(-1px);
}

.sync-both {
  background-color: #9b59b6;
  color: white;
}

.generate {
  background-color: #3498db;
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
  width: 100%;
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

/* Recherche */
.search-container {
  background: var(--card-background);
  border-radius: var(--border-radius);
  padding: var(--spacing);
  box-shadow: 0 2px 5px rgba(0,0,0,0.1);
  margin-bottom: 2rem;
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
}

/* Comments section */
.comments-section {
  margin-top: 1.5rem;
  padding-top: 1rem;
  border-top: 1px solid #ddd;
}

.comment-form {
  background: #f8f9fa;
  padding: 1rem;
  border-radius: var(--border-radius);
  margin-bottom: 1rem;
}

.add-comment {
  background-color: var(--primary-color);
  color: white;
  margin-top: 1rem;
}

/* Media queries */
@media (max-width: 768px) {
  .posts-list {
    grid-template-columns: 1fr;
  }

  .modal-content {
    width: 95%;
    padding: 1rem;
  }

  .search-actions {
    flex-direction: column;
  }

  .action-buttons {
    flex-direction: column;
  }

  .image-preview,
  .post-image {
    max-height: 200px;
  }

  .btn {
    width: 100%;
  }
}
</style>