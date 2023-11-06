
<script>
    import { onMount } from 'svelte';

    // Ceci est une variable pour stocker les articles venant de l'API.
    let articles = [];
    // Ceci est une variable pour stocker les sections venant de l'API.
    let sectionNames = {}; // { section_id: "Section Name", ... }
    let categoryNames = {}; // { category_id: "Category Name", ... }

    // Lors du chargement du composant, nous allons chercher les articles.
    onMount(async () => {
    const articlesResponse = await fetch('https://newenki.zendesk.com/api/v2/help_center/fr/articles');
    articles = (await articlesResponse.json()).articles;
    
    for (const article of articles) {
        if (!sectionNames[article.section_id]) {
            const sectionResponse = await fetch(`https://newenki.zendesk.com/api/v2/help_center/fr/sections/${article.section_id}`);
            const sectionData = await sectionResponse.json();
            sectionNames[article.section_id] = sectionData.section;
            console.log(sectionNames[article.section_id]);
        }

        let category_id = sectionNames[article.section_id].category_id; 

        if (!categoryNames[category_id]) {
            const categoryResponse = await fetch(`https://newenki.zendesk.com/api/v2/help_center/fr/categories/${category_id}`);
            const categoryData = await categoryResponse.json();
            categoryNames[article.section_id] = categoryData.category.name;
        }
    }
});




</script>

<table>
    <thead>
        <tr>
            <th>Titre</th>
            <th>Catégorie</th>
            <th>Section</th>
            <th>Label</th>
            <th>URL</th>
        </tr>
    </thead>
    <tbody>
        {#each articles as article}
            <tr>
                <!-- Affiche le titre de l'article -->
                <td>{article.title}</td>

                <td>{categoryNames[article.section_id] || article.section_id}</td>
                
                <td>{sectionNames[article.section_id] && sectionNames[article.section_id].name}</td> <!-- Nouvelle cellule pour afficher l'ID de la section -->

                <!-- Affiche le label de l'article. Si plusieurs labels, les joint par une virgule -->
                <td>{article.label_names.join(', ')}</td>
                
                <!-- Affiche une icône cliquable qui redirige vers l'URL de l'article -->
                <td>
                    <a href={article.html_url} target="_blank" rel="noopener noreferrer">
                        🔗
                    </a>
                </td>
            </tr>
        {/each}
    </tbody>
</table>

<style>
    /* Styles basiques pour le tableau */
    table {
        width: 100%;
        border-collapse: collapse;
        border-radius: 8px;
    }

    th, td {
        padding: 8px 15px;
        border: 1px solid #ddd;
    }

    th {
        background-color: #EDE5DA;
        color: #035010;
    }
</style>
