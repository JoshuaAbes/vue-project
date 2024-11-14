<script lang="ts">
import PouchDB from 'pouchdb';

// Interface pour définir la structure d'un document
interface Document {
    _id?: string;
    _rev?: string;
    title: string;
    description: string;
    createdAt: string;
}

export default {
    data() {
        return {
            documents: [] as Document[],
            storage: null as PouchDB.Database | null,
            newDocument: {
                title: '',
                description: ''
            },
            editingDocument: null as Document | null,
            showEditForm: false
        };
    },

    methods: {
        // Initialisation de la base de données
        initDatabase() {
            const db = new PouchDB('http://admim:VanilleThecat19@localhost:5984/post');
            if (db) {
                console.log("Connected to collection 'post'");
                this.storage = db;
                this.fetchDocuments(); // Charge les documents au démarrage
                this.setupSync(); // Configure la synchronisation en temps réel
            } else {
                console.warn("Something went wrong with database connection");
            }
        },

        // Configuration de la synchronisation en temps réel
        setupSync() {
            if (this.storage) {
                this.storage.changes({
                    since: 'now',
                    live: true
                }).on('change', () => {
                    this.fetchDocuments();
                });
            }
        },

        // Récupération de tous les documents
        async fetchDocuments() {
            try {
                if (this.storage) {
                    const result = await this.storage.allDocs({
                        include_docs: true
                    });
                    this.documents = result.rows.map(row => row.doc as Document);
                }
            } catch (error) {
                console.error('Error fetching documents:', error);
            }
        },

        // Génération d'un document de démonstration
        generateFakeDocument(): Document {
            return {
                title: `Document ${Math.floor(Math.random() * 1000)}`,
                description: `Description générée automatiquement ${new Date().toLocaleString()}`,
                createdAt: new Date().toISOString()
            };
        },

        // Ajout d'un document de démonstration
        async addFakeDocument() {
            try {
                if (this.storage) {
                    const fakeDoc = this.generateFakeDocument();
                    await this.storage.post(fakeDoc);
                    await this.fetchDocuments();
                }
            } catch (error) {
                console.error('Error adding fake document:', error);
            }
        },

        // Ajout d'un nouveau document à partir du formulaire
        async addDocument() {
            try {
                if (this.storage && this.newDocument.title && this.newDocument.description) {
                    const doc = {
                        ...this.newDocument,
                        createdAt: new Date().toISOString()
                    };
                    await this.storage.post(doc);
                    this.newDocument.title = '';
                    this.newDocument.description = '';
                    await this.fetchDocuments();
                }
            } catch (error) {
                console.error('Error adding document:', error);
            }
        },

        // Préparation de l'édition d'un document
        startEditing(document: Document) {
            this.editingDocument = { ...document };
            this.showEditForm = true;
        },

        // Mise à jour d'un document
        async updateDocument() {
            try {
                if (this.storage && this.editingDocument) {
                    await this.storage.put(this.editingDocument);
                    this.editingDocument = null;
                    this.showEditForm = false;
                    await this.fetchDocuments();
                }
            } catch (error) {
                console.error('Error updating document:', error);
            }
        },

        // Suppression d'un document
        async deleteDocument(document: Document) {
            try {
                if (this.storage && document._id && document._rev) {
                    await this.storage.remove(document._id, document._rev);
                    await this.fetchDocuments();
                }
            } catch (error) {
                console.error('Error deleting document:', error);
            }
        }
    },

    mounted() {
        this.initDatabase();
    }
}
</script>

<template>
    <div class="container mx-auto p-4">
        <h1 class="text-2xl font-bold mb-4">Gestion des Documents</h1>

        <!-- Formulaire d'ajout -->
        <div class="mb-6 p-4 bg-gray-100 rounded">
            <h2 class="text-xl mb-2">Ajouter un nouveau document</h2>
            <div class="flex gap-2 mb-4">
                <input v-model="newDocument.title" placeholder="Titre" class="border p-2 rounded flex-1">
                <input v-model="newDocument.description" placeholder="Description" class="border p-2 rounded flex-1">
                <button @click="addDocument" class="bg-blue-500 text-white px-4 py-2 rounded">
                    Ajouter
                </button>
            </div>
            <button @click="addFakeDocument" class="bg-green-500 text-white px-4 py-2 rounded">
                Ajouter un document de démo
            </button>
        </div>

        <!-- Liste des documents -->
        <div class="space-y-4">
            <div v-for="doc in documents" :key="doc._id" class="border p-4 rounded shadow">
                <div v-if="editingDocument && editingDocument._id === doc._id">
                    <input v-model="editingDocument.title" class="border p-2 rounded mb-2 w-full">
                    <input v-model="editingDocument.description" class="border p-2 rounded mb-2 w-full">
                    <div class="flex gap-2">
                        <button @click="updateDocument" class="bg-blue-500 text-white px-4 py-2 rounded">
                            Sauvegarder
                        </button>
                        <button @click="editingDocument = null" class="bg-gray-500 text-white px-4 py-2 rounded">
                            Annuler
                        </button>
                    </div>
                </div>
                <div v-else>
                    <h3 class="text-lg font-bold">{{ doc.title }}</h3>
                    <p class="text-gray-600">{{ doc.description }}</p>
                    <p class="text-sm text-gray-400">Créé le: {{ new Date(doc.createdAt).toLocaleString() }}</p>
                    <div class="flex gap-2 mt-2">
                        <button @click="startEditing(doc)" class="bg-yellow-500 text-white px-4 py-2 rounded">
                            Modifier
                        </button>
                        <button @click="deleteDocument(doc)" class="bg-red-500 text-white px-4 py-2 rounded">
                            Supprimer
                        </button>
                    </div>
                </div>
            </div>
        </div>
    </div>
</template>