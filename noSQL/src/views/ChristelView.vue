<script lang="ts">
import { ref } from 'vue';
import PouchDB  from 'pouchdb';


declare interface Comment {
  comment: string,
  author: string
}

declare interface Post {
  _id: string,
  _rev?: string,
  post_name: string,
  post_content: string,
  attributes: {
    creation_date: string,
    author: string
  },
  comments: Comment[]
}

export default {
  data() {
    return {
      postsData: [] as Post[],
      storage: null as PouchDB.Database | null,
      remoteDb: null as PouchDB.Database | null,
      isEditing: false,
      isAdding: false,
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
    this.fetchData();
  },

  methods: {
    initDatabase() {
      // Initialiser les bases de données locales et distantes
      this.storage = new PouchDB('commentaires_data');
      this.remoteDb = new PouchDB('http://localhost:5984/commentaires_data');

      // Configurer la synchronisation en direct
      PouchDB.sync(this.storage, this.remoteDb, {
        live: true,
        retry: true
      })
      .on('change', (info) => {
        console.log('Changement détecté dans la synchronisation :', info);
        this.fetchData(); // Actualiser les données après un changement
      })
      .on('paused', (err) => {
        console.log('Synchronisation en pause :', err || 'à jour');
      })
      .on('active', () => {
        console.log('Synchronisation active');
      })
      .on('denied', (err) => {
        console.error('Synchronisation refusée :', err);
      })
      .on('complete', (info) => {
        console.log('Synchronisation complète :', info);
      })
      .on('error', (err) => {
        console.error('Erreur de synchronisation :', err);
      });
    },

    fetchData() {
      const storage = this.storage;
      if (storage) {
        storage.allDocs({
          include_docs: true,
          attachments: true
        }).then((result: any) => {
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
          console.log('Erreur lors de la récupération des données', error);
        });
      }
    },

    startAdd() {
      this.isAdding = true;
      this.isEditing = false;
    },

    addDocument() {
      const newDoc: Post = {
        _id: new Date().toISOString(),
        post_name: this.addForm.post_name,
        post_content: this.addForm.post_content,
        attributes: {
          creation_date: new Date().toISOString(),
          author: this.addForm.attributes.author
        },
        comments: this.addForm.comments
      };
      this.putDocument(newDoc);
      this.cancelAdd();
    },

    putDocument(document: Post) {
      const db = this.storage;
      if (db) {
        db.put(document).then(() => {
          console.log('Document ajouté avec succès');
          this.fetchData();
        }).catch((error) => {
          console.log('Erreur lors de l\'ajout du document', error);
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

    addCommentEdit() {
      this.editForm.comments.push({ comment: '', author: '' });
    },

    deleteCommentEdit(index: number) {
      this.editForm.comments.splice(index, 1);
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
      const db = this.storage;
      if (db && post._id && post._rev) {
        db.remove(post._id, post._rev).then(() => {
          console.log('Document supprimé avec succès');
          this.fetchData();
        }).catch((error) => {
          console.log('Erreur lors de la suppression du document', error);
        });
      } else {
        console.log('Erreur : _id ou _rev manquant pour la suppression');
      }
    },

    saveEdit() {
      const db = this.storage;
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
          this.fetchData();
          this.cancelEdit();
        }).catch((error) => {
          console.log('Erreur lors de la mise à jour du document', error);
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
  <h1>Nombre de posts : {{ postsData.length }}</h1>
  <button @click="startAdd">Ajouter un document</button>

  <ul>
    <li v-for="post in postsData" :key="post._id">
      <div class="ucfirst">
        {{ post.post_name }}
        <em v-if="post.attributes.creation_date">- {{ post.attributes.creation_date }}</em>
      </div>
      <button @click="startEdit(post)">Modifier</button>
      <button @click="deleteDocument(post)">Supprimer</button>
    </li>
  </ul>

  <!-- Formulaire d'ajout de document -->
  <div v-if="isAdding">
    <h2>Ajouter un nouveau document</h2>
    <form @submit.prevent="addDocument">
      <label for="add_post_name">Nom du post :</label>
      <input type="text" v-model="addForm.post_name" id="add_post_name" required />

      <label for="add_post_content">Contenu du post :</label>
      <textarea v-model="addForm.post_content" id="add_post_content" required></textarea>

      <label for="add_author">Auteur :</label>
      <input type="text" v-model="addForm.attributes.author" id="add_author" required />

      <h3>Commentaires</h3>
      <div v-for="(comment, index) in addForm.comments" :key="index">
        <label :for="'comment' + index">Commentaire :</label>
        <input type="text" v-model="comment.comment" :id="'comment' + index" required />

        <label :for="'author' + index">Auteur du commentaire :</label>
        <input type="text" v-model="comment.author" :id="'author' + index" required />

        <button type="button" @click="deleteComment(index)">Supprimer le commentaire</button>
      </div>
      <button type="button" @click="addComment">Ajouter un commentaire</button>

      <button type="submit">Enregistrer</button>
      <button type="button" @click="cancelAdd">Annuler</button>
    </form>
  </div>

  <!-- Formulaire de modification de document -->
  <div v-if="isEditing">
    <h2>Modifier le document</h2>
    <form @submit.prevent="saveEdit">
      <label for="edit_post_name">Nom du post :</label>
      <input type="text" v-model="editForm.post_name" id="edit_post_name" required />

      <label for="edit_post_content">Contenu du post :</label>
      <textarea v-model="editForm.post_content" id="edit_post_content" required></textarea>

      <label for="edit_author">Auteur :</label>
      <input type="text" v-model="editForm.attributes.author" id="edit_author" required />

      <h3>Commentaires</h3>
      <div v-for="(comment, index) in editForm.comments" :key="index">
        <label :for="'edit_comment' + index">Commentaire :</label>
        <input type="text" v-model="comment.comment" :id="'edit_comment' + index" required />

        <label :for="'edit_author_comment' + index">Auteur du commentaire :</label>
        <input type="text" v-model="comment.author" :id="'edit_author_comment' + index" required />

        <button type="button" @click="deleteCommentEdit(index)">Supprimer le commentaire</button>
      </div>
      <button type="button" @click="addCommentEdit">Ajouter un commentaire</button>

      <button type="submit">Enregistrer</button>
      <button type="button" @click="cancelEdit">Annuler</button>
    </form>
  </div>
</template>
