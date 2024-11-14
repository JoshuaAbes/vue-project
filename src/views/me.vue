<script lang="ts">
import PouchDB from 'pouchdb'; // Assure-toi d'importer PouchDB

export default {

    data() {
        return {
            datas: 0, // Changement en nombre si tu veux l'utiliser comme compteur
            storage: null,
        };
    },

    methods: {

        inc() {
            this.datas++; // Incrémente le compteur
        },

        initDatabase() {
            const db = new PouchDB('http://admim:VanilleThecat19@localhost:5984/post');
            if (db) {
                console.log("Connected to collection 'post'");
            } else {
                console.warn("Something went wrong");
            }
            this.storage = db; // Stockage de l'instance PouchDB
        },

        fetchData() {
            // Ajoute la logique de récupération des données ici
        }

    },

    mounted() {
        this.initDatabase(); // Initialise la base de données lors du montage du composant
    }

}
</script>

<template>
    <h1>InfraDon2</h1>
    <p>Counter: {{ datas }}</p>
    <button type="button" role="button" @click="inc">
        +1
    </button>
</template>
