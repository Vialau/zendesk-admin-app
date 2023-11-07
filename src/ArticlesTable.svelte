
<script>
    import { onMount } from 'svelte';
    import 'bootstrap/dist/js/bootstrap.bundle.min.js'; // Assurez-vous d'avoir importé Bootstrap JS
    let selectedCategory = '';
    let selectedSection = '';
    let labelFilter = 'all'; // 'all' ou 'duplicate'

    // Ceci est une variable pour stocker les articles venant de l'API.
    let articles = [];
    // Ceci est une variable pour stocker les sections venant de l'API.
    let sectionNames = {}; // { section_id: "Section Name", ... }
    let categoryNames = {}; // { category_id: "Category Name", ... }

    let showModal = false; // Contrôle l'affichage de la modal
    let userEmail = ''; // Stocke l'email de l'utilisateur
    let userPassword = ''; // Stocke le mot de passe de l'utilisateur
    let encodedCredentials = ''; // Stocke les credentials encodées

    let labels = []; // Cet état contiendra tous les labels avec leur ID et leur nom

    let labelToDelete = null;

  // Cette fonction est appelée lors du chargement du composant pour charger les labels
  async function loadLabels() {
    try {
      const response = await fetch('https://newenki.zendesk.com/api/v2/help_center/articles/labels');
      if (response.ok) {
        labels = await response.json(); // Mettez à jour l'état des labels avec les données de l'API
      } else {
        // Gérer les erreurs ici
        console.error("Erreur lors du chargement des labels");
      }
    } catch (error) {
      console.error("Erreur lors de la connexion à l'API", error);
    }
  }

    // Pour chaque article, vous aurez besoin d'un état pour stocker le nouveau label
    let newLabels = {}; // { articleId: 'new label name' }

    // Lors du chargement du composant, nous allons chercher les articles.
    onMount(async () => {
        if (!localStorage.getItem('encodedCredentials')) {
            showModal = true; // Pas de credentials, montrez la modal
        } else {
            encodedCredentials = localStorage.getItem('encodedCredentials'); 
            await loadArticles();  
        }
    });
    
    onMount(() => {
    loadLabels(); // Appeler loadLabels lors de l'initialisation du composant
  });

    var handleLogin = async function() {
        // Encodez les credentials et stockez-les dans localStorage pour cet exemple
        encodedCredentials = btoa(`${userEmail}:${userPassword}`);
        localStorage.setItem('encodedCredentials', encodedCredentials);

        // Fermez la modal après la connexion
        showModal = false;
        await loadArticles();
    }

    var loadArticles = async function() {
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

        articles.forEach(article => {
        article.label_names.forEach(label => {
                labelCounts[label] = (labelCounts[label] || 0) + 1;
            });
        });
    }

let labelCounts = {}; // { labelName: count, ... }

// Étape 2: Ajout de Label
    async function addLabel(articleId, labelName) {
        if (labelName.trim() === '') {
            console.log("Le label ne peut pas être vide");
            return;
        }
        
        try {
      const response = await fetch(`https://newenki.zendesk.com/api/v2/help_center/articles/${articleId}/labels`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Basic ${encodedCredentials}`
        },
        body: JSON.stringify({ label: { name: labelName } })
      });

      if (response.ok) {
        // Si le label est ajouté avec succès, mettez à jour l'interface utilisateur
        let newLabel = await response.json(); // Supposons que l'API renvoie le nouveau label
        articles = articles.map((article) => {
          if (article.id === articleId) {
            // Ajoutez le nouveau label à la liste des labels
            return { ...article, label_names: [...article.label_names, newLabel] };
          }
          return article;
        });
      } else {
        // Gérer les erreurs de l'API ici
        console.error("Erreur lors de l'ajout du label");
      }
    } catch (error) {
      console.error("Erreur lors de la connexion à l'API", error);
    }

    // Cachez l'input et réinitialisez l'état d'édition
    editingLabel[articleId] = false;
    // Il est important de mettre à jour Svelte sur les changements d'état
    editingLabel = { ...editingLabel };
  }


    let editingLabel = {}; // { articleId: boolean }

    function promptDeleteLabel(articleId, labelName) {
    labelToDelete = { articleId, labelName };
    const deleteModal = new bootstrap.Modal(document.getElementById('deleteLabelModal'));
    deleteModal.show();
  }

  async function confirmDelete() {
    if (labelToDelete) {
      await deleteLabel(labelToDelete.articleId, labelToDelete.labelName);
      // Vous pourriez vouloir cacher la modal après la suppression
      const deleteModal = bootstrap.Modal.getInstance(document.getElementById('deleteLabelModal'));
      deleteModal.hide();
    }
  }

    async function deleteLabel(articleId, labelName) {
    const label = labels.find(l => l.name === labelName);
    if (!label) {
      console.error("Label non trouvé");
      return;
    }

    try {
      const response = await fetch(`https://newenki.zendesk.com/api/v2/help_center/articles/${articleId}/labels/${label.id}`, {
        method: 'DELETE',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Basic ${encodedCredentials}`
        },
      });

      if (response.ok) {
        // Supprimez le label de l'affichage après la suppression réussie
        articles = articles.map(article => {
          if (article.id === articleId) {
            return {
              ...article,
              label_names: article.label_names.filter(name => name !== labelName)
            };
          }
          return article;
        });
      } else {
        // Gérer les erreurs de l'API ici
        console.error("Erreur lors de la suppression du label");
      }
    } catch (error) {
      console.error("Erreur lors de la connexion à l'API", error);
    }
  }


</script>

{#if showModal}
<div class="modal">
  <div class="modal-content">
    <h2>Connexion</h2>
    <input type="email" bind:value={userEmail} placeholder="Email">
    <input type="password" bind:value={userPassword} placeholder="Mot de passe">
    <button on:click={handleLogin}>Connexion</button>
  </div>
</div>
{:else}


<table>
    <thead>
        <tr>
            <th>Titre de l'article</th>
            <th>
                <select bind:value={selectedCategory}>
                    <option value="">Catégories</option>
                    {#each Object.values(categoryNames) as category}
                        <option value={category}>{category}</option>
                    {/each}
                </select>
            </th>
            <th>
                <select bind:value={selectedSection}>
                    <option value="">Sections</option>
                    {#each Object.values(sectionNames) as section}
                        <option value={section.name}>{section.name}</option>
                    {/each}
                </select>
                <th>
                    <select bind:value={labelFilter}>
                        <option value="all">Tous les Labels</option>
                        <option value="duplicate">Doublons</option>
                    </select>
                </th>
            <th>URL</th>
        </tr>
    </thead>
    <tbody>
        {#each articles.filter(article => 
            (labelFilter === 'all' || 
            (labelFilter === 'duplicate' && article.label_names.some(label => labelCounts[label] > 1))) &&
            (!selectedCategory || categoryNames[article.section_id] === selectedCategory) && 
            (!selectedSection || (sectionNames[article.section_id] && sectionNames[article.section_id].name === selectedSection))
        ) as article}
            <tr>
                <!-- Affiche le titre de l'article -->
                <td>{article.title}</td>

                <td>{categoryNames[article.section_id] || article.section_id}</td>
                
                <td>{sectionNames[article.section_id] && sectionNames[article.section_id].name}</td> <!-- Nouvelle cellule pour afficher l'ID de la section -->

                <!-- Affiche le label de l'article. Si plusieurs labels, les joint par une virgule -->
                <td>
                    {#each article.label_names as label, i (label)}
                      <!-- Affichage des labels existants -->
                      <span class={labelCounts[label] > 1 ? 'badge rounded-pill text-bg-danger' : 'badge rounded-pill text-bg-success'}>
                        {label}
                        <!-- Supposons que vous avez une fonction removeLabel à implémenter pour gérer la suppression -->
                        <i class="bi bi-trash" on:click={() => promptDeleteLabel(article.id, label)}></i>
                      </span>
                      {#if i !== article.label_names.length - 1}
                        ,
                      {/if}
                    {/each}
                    {#if article.label_names.length === 0}
                      <!-- Placeholder si aucun label -->
                      <span class="badge rounded-pill text-bg-light">Aucun label</span>
                    {/if}
                    <!-- Ajout d'un nouveau label -->
                    <!-- svelte-ignore a11y-click-events-have-key-events -->
                    <span class="badge rounded-pill text-bg-secondary" on:click={() => (editingLabel[article.id] = true)}>
                      <i class="bi bi-plus-lg"></i> Ajouter un label
                    </span>
                    {#if editingLabel[article.id]}
                      <input 
                        class="form-control" 
                        type="text" 
                        placeholder="Nouveau label" 
                        on:keydown={(e) => e.key === 'Enter' && addLabel(article.id, e.target.value)}
                        on:blur={() => (editingLabel[article.id] = false)}
                      />
                    {/if}
                  </td>

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
{/if}

<style>

.bi-plus-lg {
    margin-right: 5px;
  }

  input.form-control {
    width: auto; /* Adjust as necessary */
    display: inline-block; /* Align with other inline elements */
    margin-top: 5px; /* Provide some spacing */
  }
i.bi-trash {
    cursor: pointer;
    color: white;
  }
  
  .btn-sm {
    padding: 0; /* Remove padding to make it align better */
    border: none; /* Optional: removes border */
    background: none; /* Optional: removes default button background */
  }

.modal {
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background-color: rgba(0, 0, 0, 0.5);
    display: flex;
    justify-content: center;
    align-items: center;
    z-index: 2;
  }
  .modal-content {
    background: white;
    padding: 20px;
    border-radius: 5px;
    text-align: center;
  }
  input {
    display: block;
    margin-bottom: 10px;
    padding: 10px;
    width: calc(100% - 20px);
  }
  button {
    padding: 10px 20px;
    cursor: pointer;
  }

    /* Styles basiques pour le tableau */
    table {
        width: 100%;
        border-collapse: collapse;
        border-radius: 8px;
        background-color: white;
    }

    th, td {
        padding: 8px 15px;
        border: 1px solid #ddd;
    }

    th {
        background-color: white;
        color: black;
    }

    th select {
        border-style: none;
        color: black;
        font-weight: bold; 
    }

    .label {

    }

</style>
