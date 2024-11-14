<template>
  <div class="ex">
    <h1>This is my page</h1>
    <div class="controls">
      <button @click="getAllDocs">Voir documents ({{ btnValue }})</button>
      <button @click="addFakeDoc">Ajouter un document</button>
      <button @click="updateRandomDoc">Modifier un document</button>
      <button @click="deleteRandomDoc">Supprimer un document</button>
    </div>

    <div class="documents" v-if="documents.length">
      <div v-for="doc in documents" :key="doc._id" class="document-item">
        <h3>{{ doc.title }}</h3>
        <p>{{ doc.content }}</p>
        <p>Créé le: {{ doc.createdAt }}</p>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  data() {
    return {
      btnValue: 0,
      db: null,
      documents: []
    }
  },
  methods: {
    addToBtn() {
      this.btnValue += 1
    },

    initDatabase() {
      try {
        this.db = new PouchDB('http://admin:VanilleThecat19@127.0.0.1:5984/post')
        console.log('Base de données initialisée :', this.db)
      } catch (error) {
        console.error("Erreur lors de l'initialisation de la base de données :", error)
      }
    },

    generateFakeDoc() {
      const timestamp = new Date().toISOString()
      return {
        _id: `doc_${timestamp}`,
        title: `Article ${Math.floor(Math.random() * 1000)}`,
        content: `Ceci est le contenu de l'article ${Math.random().toString(36).substring(7)}`,
        createdAt: timestamp,
        type: 'article'
      }
    },

    getAllDocs() {
      if (!this.db) {
        console.error('Database not initialized')
        return
      }

      this.db
        .allDocs({
          include_docs: true,
          attachments: true
        })
        .then((result) => {
          this.documents = result.rows.map((row) => row.doc)
          this.addToBtn()
        })
        .catch((error) => {
          console.error('Error fetching docs:', error)
        })
    },

    async addFakeDoc() {
      if (!this.db) return

      try {
        const doc = this.generateFakeDoc()
        await this.db.put(doc)
        console.log('Document added successfully')
        this.getAllDocs()
      } catch (error) {
        console.error('Error adding document:', error)
      }
    },

    async updateRandomDoc() {
      if (!this.db || this.documents.length === 0) return

      try {
        const docToUpdate = this.documents[Math.floor(Math.random() * this.documents.length)]
        const updatedDoc = {
          ...docToUpdate,
          title: `Article modifié ${Math.floor(Math.random() * 1000)}`,
          content: `Contenu modifié ${Math.random().toString(36).substring(7)}`,
          updatedAt: new Date().toISOString()
        }

        await this.db.put(updatedDoc)
        console.log('Document updated successfully')
        this.getAllDocs()
      } catch (error) {
        console.error('Error updating document:', error)
      }
    },

    async deleteRandomDoc() {
      if (!this.db || this.documents.length === 0) return

      try {
        const docToDelete = this.documents[Math.floor(Math.random() * this.documents.length)]
        await this.db.remove(docToDelete)
        console.log('Document deleted successfully')
        this.getAllDocs()
      } catch (error) {
        console.error('Error deleting document:', error)
      }
    }
  },
  mounted() {
    this.initDatabase()
    this.getAllDocs()
  }
}
</script>

<style>
.ex {
  padding: 20px;
}

.controls {
  margin: 20px 0;
  display: flex;
  gap: 10px;
}

.documents {
  margin-top: 20px;
}

.document-item {
  border: 1px solid #ddd;
  padding: 15px;
  margin-bottom: 10px;
  border-radius: 4px;
}

button {
  padding: 8px 16px;
  background-color: #4caf50;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

button:hover {
  background-color: #45a049;
}

@media (min-width: 1024px) {
  .ex {
    min-height: 100vh;
  }
}
</style>
