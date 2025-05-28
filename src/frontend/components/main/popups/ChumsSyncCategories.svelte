<script lang="ts">
    import { activePopup, categories, chumsSyncCategories, shows } from "../../../stores"
    import { sortByName } from "../../../components/helpers/array"
    import Icon from "../../../components/helpers/Icon.svelte"
    import T from "../../../components/helpers/T.svelte"
    import Button from "../../../components/inputs/Button.svelte"
    import Checkbox from "../../../components/inputs/Checkbox.svelte"

    $: categoryOptions = sortByName(
        Object.entries($categories).map(([id, category]) => ({
            id,
            name: category.name,
            count: Object.values($shows).filter((show) => show.category === id).length,
        }))
    )

    function handleChange(e: Event, id: string) {
        const customEvent = e as CustomEvent<boolean>
        toggleCategory(id, customEvent.detail)
    }

    function toggleCategory(id: string, checked: boolean) {
        if (checked) {
            chumsSyncCategories.update((categories) => [...categories, id])
        } else {
            chumsSyncCategories.update((categories) => categories.filter((c) => c !== id))
        }
    }
</script>

<div class="popup">
    <div class="header">
        <h2><T id="chums.sync_categories" /></h2>
        <button class="close" on:click={() => (activePopup.set(null))}>
            <Icon id="close" />
        </button>
    </div>

    <div class="content">
        <p><T id="chums.sync_categories_description" /></p>

        <div class="categories">
            {#each categoryOptions as { id, name, count }}
                <div class="category">
                    <Checkbox
                        checked={$chumsSyncCategories.includes(id)}
                        on:change={(e) => handleChange(e, id)}
                    >
                        <T id={name} />
                    </Checkbox>
                    <span class="count">({count})</span>
                </div>
            {/each}
        </div>

        <div class="buttons">
            <Button on:click={() => (activePopup.set(null))}>
                <T id="actions.done" />
            </Button>
        </div>
    </div>
</div>

<style>
    .popup {
        width: 400px;
        max-width: 90vw;
    }

    .header {
        display: flex;
        align-items: center;
        justify-content: space-between;
        margin-bottom: 1rem;
    }

    .content {
        display: flex;
        flex-direction: column;
        gap: 1rem;
    }

    .categories {
        display: flex;
        flex-direction: column;
        gap: 0.5rem;
        max-height: 300px;
        overflow-y: auto;
    }

    .category {
        display: flex;
        align-items: center;
        gap: 0.5rem;
    }

    .count {
        color: var(--color-text-secondary);
        font-size: 0.9em;
    }

    .buttons {
        display: flex;
        justify-content: flex-end;
        margin-top: 1rem;
    }
</style> 