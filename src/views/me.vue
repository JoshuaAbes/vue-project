<script lang="ts">
import PouchDB from 'pouchdb';
import PouchDBFind from 'pouchdb-find';
import { defineComponent } from 'vue';
PouchDB.plugin(PouchDBFind);

/**
 * Interface définissant la structure d'un document
 * @interface Document
 * @property {string} [_id] - Identifiant unique du document
 * @property {string} [_rev] - Numéro de révision du document
 * @property {string} title - Titre du document
 * @property {string} description - Description du document
 * @property {string} createdAt - Date de création
 * @property {Object} [_attachments] - Pièces jointes du document
 */
interface Document {
    _id?: string;
    _rev?: string;
    title: string;
    description: string;
    createdAt: string;
    _attachments?: {
        [key: string]: {
            content_type: string;
            data: string;
        };
    };
}

export default defineComponent({
    /**
     * État initial du composant
     * @returns {Object} État initial
     */
    data() {
        return {
            documents: [] as Document[],          // Liste des documents
            storage: null as PouchDB.Database | null,    // Base de données distante
            localDb: null as PouchDB.Database | null,    // Base de données locale
            newDocument: {                        // Nouveau document à créer
                title: '',
                description: ''
            },
            editingDocument: null as Document | null,    // Document en cours d'édition
            showEditForm: false,                  // Affichage du formulaire d'édition
            syncStatus: '',                       // Statut de synchronisation
            isSyncing: false,                     // État de synchronisation en cours
            searchQuery: '',                      // Terme de recherche
            batchSize: 10,                        // Nombre de documents à générer
            selectedFiles: [] as File[],          // Fichiers sélectionnés
            documentAttachments: {} as { [key: string]: string[] },  // Pièces jointes par document
            imageUrls: {} as { [key: string]: string },  // URLs des images
        };
    },

    methods: {
        /**
         * Initialise les bases de données et configure la réplication
         */
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

        /**
         * Crée un index sur le champ 'title' pour la recherche
         */
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

        /**
         * Génère un lot de documents de test
         */
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

        /**
         * Effectue une recherche de documents par titre
         */
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

        /**
         * Déclenche une synchronisation manuelle avec la base distante
         */
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

        /**
         * Récupère tous les documents et leurs pièces jointes
         */
        async fetchDocuments() {
            try {
                if (this.localDb) {
                    const result = await this.localDb.allDocs({
                        include_docs: true
                    });
                    this.documents = result.rows.map(row => row.doc as Document);

                    // Récupérer les pièces jointes pour chaque document
                    for (const doc of this.documents) {
                        if (doc._id) {
                            await this.fetchDocumentAttachments(doc._id);
                            // Charger les URLs des images pour chaque pièce jointe
                            if (this.documentAttachments[doc._id]) {
                                for (const attachment of this.documentAttachments[doc._id]) {
                                    if (this.isImage(attachment)) {
                                        await this.loadImageUrl(doc._id, attachment);
                                    }
                                }
                            }
                        }
                    }
                }
            } catch (error) {
                console.error('Error fetching documents:', error);
            }
        },

        /**
         * Génère un document avec des données aléatoires
         * @returns {Document} Document généré
         */
        generateFakeDocument(): Document {
            return {
                title: `Document ${Math.floor(Math.random() * 1000)}`,
                description: `Description générée automatiquement ${new Date().toLocaleString()}`,
                createdAt: new Date().toISOString()
            };
        },

        /**
         * Gère la sélection des fichiers à attacher
         * @param {Event} event - Événement de sélection de fichier
         * @param {string} [documentId] - ID du document si ajout à un document existant
         */
        async handleFileSelect(event: Event, documentId?: string) {
            const input = event.target as HTMLInputElement;
            if (!input.files?.length) return;

            const files = Array.from(input.files);
            if (documentId) {
                // Ajouter des fichiers à un document existant
                await this.addAttachmentsToDocument(documentId, files);
            } else {
                // Pour un nouveau document
                this.selectedFiles = files;
            }
        },

        /**
         * Ajoute des pièces jointes à un document
         * @param {string} docId - ID du document
         * @param {File[]} files - Fichiers à attacher
         */
        async addAttachmentsToDocument(docId: string, files: File[]) {
            try {
                if (!this.localDb) return;

                const doc = await this.localDb.get(docId);

                for (const file of files) {
                    const reader = new FileReader();
                    reader.onloadend = async () => {
                        try {
                            const base64Data = reader.result as string;
                            const base64Content = base64Data.split(',')[1];
                            console.log('Ajout de la pièce jointe:', file.name); // Log pour debug

                            await this.localDb!.putAttachment(
                                docId,
                                file.name,
                                doc._rev,
                                base64Content,
                                file.type
                            );

                            await this.fetchDocumentAttachments(docId);
                        } catch (err) {
                            console.error('Erreur lors de l\'ajout de la pièce jointe:', err);
                        }
                    };
                    reader.readAsDataURL(file);
                }
            } catch (error) {
                console.error('Erreur lors de l\'ajout des pièces jointes:', error);
            }
        },

        // Supprimer une pièce jointe
        async deleteAttachment(docId: string, attachmentName: string) {
            try {
                if (!this.localDb) return;

                const doc = await this.localDb.get(docId);
                await this.localDb.removeAttachment(docId, attachmentName, doc._rev);
                await this.fetchDocumentAttachments(docId);
            } catch (error) {
                console.error('Erreur lors de la suppression de la pièce jointe:', error);
            }
        },

        // Récupérer les pièces jointes d'un document
        async fetchDocumentAttachments(docId: string) {
            try {
                if (!this.localDb) return;

                const doc = await this.localDb.get(docId, { attachments: true });
                console.log('Document avec attachements:', doc);
                if (doc._attachments) {
                    this.documentAttachments[docId] = Object.keys(doc._attachments);
                    console.log('Attachements pour', docId, ':', this.documentAttachments[docId]);
                }
            } catch (error) {
                console.error('Erreur lors de la récupération des pièces jointes:', error);
            }
        },

        // Ajout d'un nouveau document avec pièces jointes
        async addDocument() {
            try {
                if (this.localDb && this.newDocument.title && this.newDocument.description) {
                    const doc = {
                        ...this.newDocument,
                        createdAt: new Date().toISOString()
                    };
                    const response = await this.localDb.post(doc);

                    if (this.selectedFiles.length > 0) {
                        await this.addAttachmentsToDocument(response.id, this.selectedFiles);
                    }

                    this.newDocument.title = '';
                    this.newDocument.description = '';
                    this.selectedFiles = [];
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
        },

        /**
         * Vérifie si un fichier est une image
         * @param {string} filename - Nom du fichier
         * @returns {boolean} True si c'est une image
         */
        isImage(filename: string): boolean {
            const imageExtensions = ['.jpg', '.jpeg', '.png', '.gif', '.webp'];
            return imageExtensions.some(ext => filename.toLowerCase().endsWith(ext));
        },

        handleImageError(event: Event) {
            const img = event.target as HTMLImageElement;
            console.error('Erreur de chargement de l\'image:', img.src);
            // Essayons de charger directement depuis la base de données locale
            if (this.localDb && img.dataset.docId && img.dataset.attachment) {
                this.localDb.getAttachment(img.dataset.docId, img.dataset.attachment)
                    .then(blob => {
                        if (blob && blob instanceof Blob) {
                            img.src = URL.createObjectURL(blob);
                        }
                    })
                    .catch(err => console.error('Erreur lors du chargement local:', err));
            }
        },

        getAttachmentUrl(docId: string, attachmentName: string): string {
            const credentials = btoa('admim:VanilleThecat19'); // Encodage Base64 des credentials
            return `http://localhost:5984/post/${docId}/${attachmentName}`;
        },

        /**
         * Récupère les données d'une image
         * @param {string} docId - ID du document
         * @param {string} attachmentName - Nom de la pièce jointe
         * @returns {Promise<string|null>} URL de l'image
         */
        async getImageData(docId: string, attachmentName: string) {
            try {
                if (!this.localDb) return null;
                const attachment = await this.localDb.getAttachment(docId, attachmentName);
                if (attachment instanceof Blob) {
                    return URL.createObjectURL(attachment);
                }
                // Si c'est un Buffer, le convertir en Blob
                if (attachment) {
                    const blob = new Blob([attachment], { type: 'image/*' });
                    return URL.createObjectURL(blob);
                }
                return null;
            } catch (error) {
                console.error('Erreur lors de la récupération de l\'image:', error);
                return null;
            }
        },

        async loadImageUrl(docId: string, attachment: string) {
            this.imageUrls[`${docId}-${attachment}`] = await this.getImageData(docId, attachment) || '';
        }
    },

    /**
     * Hooks de cycle de vie
     * Initialise la base de données et crée l'index au montage du composant
     */
    mounted() {
        this.initDatabase();
        this.createIndex();
    }

})
</script>

<template>
    <div class="container mx-auto p-4 overflow-x-hidden">
        <!-- En-tête avec bouton de synchronisation -->
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

        <!-- Section Factory et Recherche -->
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

        <!-- Formulaire d'ajout de document -->
        <div class="mb-6 p-4 bg-gray-100 rounded">
            <h2 class="text-xl mb-2">Ajouter un nouveau document</h2>
            <div class="flex gap-2 mb-4">
                <input v-model="newDocument.title" placeholder="Titre" class="border p-2 rounded flex-1">
                <input v-model="newDocument.description" placeholder="Description" class="border p-2 rounded flex-1">
                <button @click="addDocument" class="bg-blue-500 text-white px-4 py-2 rounded">
                    Ajouter
                </button>
            </div>
            <!-- Nouveau : Section pour l'ajout de médias -->
            <div class="mb-4">
                <label class="block text-sm font-medium text-gray-700 mb-2">
                    Ajouter des médias
                </label>
                <input type="file" multiple @change="handleFileSelect" class="border p-2 rounded w-full">
                <div v-if="selectedFiles.length" class="mt-2">
                    <p class="text-sm text-gray-600">
                        Fichiers sélectionnés :
                        {{ selectedFiles.map(f => f.name).join(', ') }}
                    </p>
                </div>
            </div>
            <button @click="generateFakeDocument" class="bg-green-500 text-white px-4 py-2 rounded">
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

                    <!-- Nouveau : Section des médias attachés -->
                    <div class="mt-4">
                        <h4 class="text-sm font-medium">Médias attachés :</h4>
                        <div v-if="documentAttachments[doc._id!]" class="flex flex-wrap gap-2 mt-2">
                            <div v-for="attachment in documentAttachments[doc._id!]" :key="attachment" class="relative">
                                <template v-if="isImage(attachment)">
                                    <img :src="imageUrls[`${doc._id}-${attachment}`]"
                                        style="max-width: 300px; max-height: 300px; object-fit: contain;"
                                        class="rounded" :alt="attachment">
                                    <button @click="deleteAttachment(doc._id!, attachment)"
                                        class="absolute top-0 right-0 bg-red-500 text-white p-1 rounded-full text-xs">
                                        ×
                                    </button>
                                </template>
                                <div v-else class="text-sm">
                                    {{ attachment }}
                                </div>
                            </div>
                        </div>
                        <div class="mt-2">
                            <input type="file" multiple @change="e => handleFileSelect(e, doc._id)"
                                class="border p-2 rounded w-full">
                        </div>
                    </div>

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

<style>
body {
    overflow-x: hidden;
}
</style>