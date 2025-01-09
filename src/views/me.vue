<script lang="ts">
import PouchDB from 'pouchdb';
import PouchDBFind from 'pouchdb-find';
PouchDB.plugin(PouchDBFind);

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
            localDb: null as PouchDB.Database | null,  // Base de données locale pour la réplication
            newDocument: {
                title: '',
                description: ''
            },
            editingDocument: null as Document | null,
            showEditForm: false,
            syncStatus: '',  // Pour afficher le statut de synchronisation
            isSyncing: false, // Pour gérer l'état du bouton pendant la synchronisation
            searchQuery: '',  // Pour la recherche
            batchSize: 10,   // Nombre de documents à générer
        };
    },

    methods: {
        // Initialisation des bases de données (locale et distante) avec réplication
        initDatabase() {
            // Création de la base de données locale
            this.localDb = new PouchDB('local_posts');

            // Connexion à la base de données distante
            const remoteDb = new PouchDB('http://admim:VanilleThecat19@localhost:5984/post');

            if (remoteDb) {
                console.log("Connected to remote database 'post'");
                this.storage = remoteDb;

                // Configuration de la réplication bidirectionnelle en temps réel
                PouchDB.sync(this.localDb, remoteDb, {
                    live: true, // Synchronisation continue
                    retry: true // Réessayer en cas d'échec
                }).on('change', (change) => {
                    // Déclenché lors d'un changement dans l'une des bases
                    console.log('Changement détecté:', change);
                    this.fetchDocuments();
                }).on('error', (err) => {
                    console.error('Erreur de synchronisation:', err);
                });

                this.fetchDocuments(); // Chargement initial des documents
            }
        },

        // Création de l'index
        async createIndex() {
            try {
                if (this.localDb) {
                    await this.localDb.createIndex({
                        index: {
                            fields: ['title'],
                            name: 'title_index'
                        }
                    });
                    console.log('Index créé avec succès');
                }
            } catch (error) {
                console.error('Erreur lors de la création de l\'index:', error);
            }
        },

        // Factory pour générer plusieurs documents
        async generateBatchDocuments() {
            try {
                if (this.localDb) {
                    const batch = [];
                    for (let i = 0; i < this.batchSize; i++) {
                        batch.push({
                            title: `Document Test ${Math.floor(Math.random() * 1000)}`,
                            description: `Description générée ${i} - ${new Date().toLocaleString()}`,
                            createdAt: new Date().toISOString()
                        });
                    }

                    await this.localDb.bulkDocs(batch);
                    await this.fetchDocuments();
                    console.log(`${this.batchSize} documents générés`);
                }
            } catch (error) {
                console.error('Erreur lors de la génération des documents:', error);
            }
        },

        // Recherche par titre
        async searchDocuments() {
            try {
                if (this.localDb && this.searchQuery) {
                    const result = await this.localDb.find({
                        selector: {
                            title: { $regex: RegExp(this.searchQuery, 'i') }
                        }
                    });
                    this.documents = result.docs as Document[];
                } else {
                    // Si la recherche est vide, afficher tous les documents
                    await this.fetchDocuments();
                }
            } catch (error) {
                console.error('Erreur lors de la recherche:', error);
            }
        },

        // Synchronisation manuelle
        async synchronizeManually() {
            try {
                if (!this.localDb || !this.storage || this.isSyncing) return;

                this.isSyncing = true;
                this.syncStatus = 'Synchronisation en cours...';

                await PouchDB.sync(this.localDb, this.storage, {
                    retry: true
                });

                await this.fetchDocuments();
                this.syncStatus = 'Synchronisation réussie !';

                setTimeout(() => {
                    this.syncStatus = '';
                }, 3000);
            } catch (error) {
                console.error('Erreur de synchronisation:', error);
                this.syncStatus = 'Erreur de synchronisation';
            } finally {
                this.isSyncing = false;
            }
        },

        // Récupération de tous les documents
        async fetchDocuments() {
            try {
                if (this.localDb) {
                    const result = await this.localDb.allDocs({
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
                if (this.localDb) {
                    const fakeDoc = this.generateFakeDocument();
                    await this.localDb.post(fakeDoc);
                    await this.fetchDocuments();
                }
            } catch (error) {
                console.error('Error adding fake document:', error);
            }
        },

        // Ajout d'un nouveau document
        async addDocument() {
            try {
                if (this.localDb && this.newDocument.title && this.newDocument.description) {
                    const doc = {
                        ...this.newDocument,
                        createdAt: new Date().toISOString()
                    };
                    await this.localDb.post(doc);
                    this.newDocument.title = '';
                    this.newDocument.description = '';
                    await this.fetchDocuments();
                }
            } catch (error) {
                console.error('Error adding document:', error);
            }
        },

        // Préparation de l'édition
        startEditing(document: Document) {
            this.editingDocument = { ...document };
            this.showEditForm = true;
        },

        // Mise à jour d'un document
        async updateDocument() {
            try {
                if (this.localDb && this.editingDocument) {
                    await this.localDb.put(this.editingDocument);
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
                if (this.localDb && document._id && document._rev) {
                    await this.localDb.remove(document._id, document._rev);
                    await this.fetchDocuments();
                }
            } catch (error) {
                console.error('Error deleting document:', error);
            }
        }
    },

    mounted() {
        this.initDatabase();
        this.createIndex();  // Création de l'index au démarrage
    }
}
</script>

<template>
    <div class="container mx-auto p-4">
        <div class="flex justify-between items-center mb-4">
            <h1 class="text-2xl font-bold">Gestion des Documents</h1>
            <div class="flex items-center gap-4">
                <span class="text-sm" :class="{
                    'text-green-600': syncStatus === 'Synchronisation réussie !',
                    'text-blue-600': syncStatus === 'Synchronisation en cours...',
                    'text-red-600': syncStatus === 'Erreur de synchronisation'
                }">{{ syncStatus }}</span>
                <button @click="synchronizeManually" :disabled="isSyncing"
                    class="bg-purple-500 text-white px-4 py-2 rounded flex items-center gap-2 disabled:opacity-50">
                    <span v-if="isSyncing">Synchronisation...</span>
                    <span v-else>Synchroniser maintenant</span>
                </button>
            </div>
        </div>

        <!-- Factory et Recherche -->
        <div class="mb-6 p-4 bg-gray-100 rounded">
            <div class="flex gap-4 mb-4">
                <div class="flex-1">
                    <label class="block text-sm font-medium text-gray-700 mb-2">
                        Générer des documents de test
                    </label>
                    <div class="flex gap-2">
                        <input v-model="batchSize" type="number" min="1" class="border p-2 rounded w-24">
                        <button @click="generateBatchDocuments" class="bg-indigo-500 text-white px-4 py-2 rounded">
                            Générer
                        </button>
                    </div>
                </div>
                <div class="flex-1">
                    <label class="block text-sm font-medium text-gray-700 mb-2">
                        Rechercher par titre
                    </label>
                    <input v-model="searchQuery" @input="searchDocuments" type="text" placeholder="Rechercher..."
                        class="border p-2 rounded w-full">
                </div>
            </div>
        </div>

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