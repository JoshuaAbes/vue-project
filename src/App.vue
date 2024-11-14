<script setup lang="ts">
<<<<<<< HEAD
import { RouterLink, RouterView } from 'vue-router'
import HelloWorld from './components/HelloWorld.vue'
import PouchDB from 'pouchdb';

// Méthode pour créer une base de données
function createDatabase() {
  // Créer une base de données locale nommée "kittens"
  const db = new PouchDB('kittens');

  // Ajouter un document dans la base de données
  const doc = {
    _id: '001',
    name: 'Mittens',
    species: 'Cat',
    age: 3
  };

  db.put(doc)
    .then(() => {
      console.log('Document ajouté avec succès !');
    })
    .catch(err => {
      console.error('Erreur lors de l’ajout du document :', err);
    });
}
=======
import { RouterLink, RouterView } from 'vue-router';
import HelloWorld from './components/HelloWorld.vue';
import { onMounted } from 'vue'; // Importe le hook onMounted de Vue
import PouchDB from 'Pouchdb';
>>>>>>> db0606d37123eb15146347d02bbf65004facc38c
</script>

<template>
  <header>
    <img alt="Vue logo" class="logo" src="@/assets/logo.svg" width="125" height="125" />

    <div class="wrapper">
      <HelloWorld msg="You did it!" />

      <nav>
        <RouterLink to="/">Home</RouterLink>
        <RouterLink to="/about">About</RouterLink>
<<<<<<< HEAD
        <RouterLink to="/me">me</RouterLink>
=======
        <RouterLink to="/JoshuaView">JoshuaView</RouterLink>
>>>>>>> db0606d37123eb15146347d02bbf65004facc38c
      </nav>
    </div>
  </header>

  <RouterView />
</template>

<style scoped>
header {
  line-height: 1.5;
  max-height: 100vh;
}

.logo {
  display: block;
  margin: 0 auto 2rem;
}

nav {
  width: 100%;
  font-size: 12px;
  text-align: center;
  margin-top: 2rem;
}

nav a.router-link-exact-active {
  color: var(--color-text);
}

nav a.router-link-exact-active:hover {
  background-color: transparent;
}

nav a {
  display: inline-block;
  padding: 0 1rem;
  border-left: 1px solid var(--color-border);
}

nav a:first-of-type {
  border: 0;
}

@media (min-width: 1024px) {
  header {
    display: flex;
    place-items: center;
    padding-right: calc(var(--section-gap) / 2);
  }

  .logo {
    margin: 0 2rem 0 0;
  }

  header .wrapper {
    display: flex;
    place-items: flex-start;
    flex-wrap: wrap;
  }

  nav {
    text-align: left;
    margin-left: -1rem;
    font-size: 1rem;
    padding: 1rem 0;
    margin-top: 1rem;
  }
}
</style>
