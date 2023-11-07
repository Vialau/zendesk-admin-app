<!-- src/App.svelte -->

<script>
    import ArticlesTable from './ArticlesTable.svelte';
    import 'bootstrap/dist/css/bootstrap.min.css';
    import 'bootstrap';
    import 'jquery';
    import 'popper.js';
</script>

<main>
    <h1>Administration des articles Zendesk</h1>
    <ArticlesTable />
</main>

<style>
    /* Styles basiques pour le conteneur principal */
    main {
        margin: auto;
        padding: 0 20px;
        box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
    }
</style>
