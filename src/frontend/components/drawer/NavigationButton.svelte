<script lang="ts">
    import { actions, audioFolders, audioPlaylists, categories, dictionary, drawerTabsData, mediaFolders, notFound, overlayCategories, scriptures, templateCategories } from "../../stores"
    import { getActionIcon } from "../actions/actions"
    import Icon from "../helpers/Icon.svelte"
    import T from "../helpers/T.svelte"
    import Button from "../inputs/Button.svelte"
    import HiddenInput from "../inputs/HiddenInput.svelte"

    export let id: string
    export let category: any
    export let length: { [key: string]: number }
    export let categoryId = ""
    export let isSubmenu = false

    function setTab(tabID: string) {
        drawerTabsData.update((dt) => {
            dt[id] = { ...(dt[id] ? dt[id] : {}), activeSubTab: isSubmenu ? categoryId : tabID, enabled: dt[id]?.enabled !== false }

            if (isSubmenu) dt[id].activeSubmenu = tabID
            else delete dt[id].activeSubmenu

            return dt
        })

        if (category.openTrigger) category.openTrigger(tabID)
    }

    const nameCategories = {
        shows: (c: { id: string; name: string }) => categories.update((a) => setName(a, c)),
        media: (c: { id: string; name: string }) => mediaFolders.update((a) => setName(a, c)),
        audio: (c: { id: string; name: string }) => audioFolders.update((a) => setName(a, c)),
        playlist: (c: { id: string; name: string }) => audioPlaylists.update((a) => setName(a, c)),
        overlays: (c: { id: string; name: string }) => overlayCategories.update((a) => setName(a, c)),
        templates: (c: { id: string; name: string }) => templateCategories.update((a) => setName(a, c)),
        scripture: (c: { id: string; name: string }) => scriptures.update((a) => setName(a, c, "customName"))
    }
    const setName = <T,>(a: T, { name, id }, nameKey = "name"): T => {
        // api scriptures
        if (!a[id]) id = Object.entries(a as any).find(([_, a]: any) => a.id === id)?.[0]
        if (!a[id]) return a

        if (a[id].default) delete a[id].default
        a[id][nameKey] = name

        return a
    }

    function changeName(e: any, categoryId: string) {
        if ($audioPlaylists[categoryId]) id = "playlist"
        if (nameCategories[id]) nameCategories[id]({ name: e.detail.value, id: categoryId })
        else console.log("rename " + id)
    }

    let editActive = false

    $: red = id === "scripture" && $notFound.bible.find((a) => a.id === category.id)

    const defaultFolders = ["all", "unlabeled", "favourites", "effects_library", "effects", "online", "screens", "cameras", "microphones", "audio_streams"]
    const tabsWithCategories = ["shows", "media", "audio", "overlays", "templates", "scripture"]
</script>

<!-- #category_${id}_button__category_${id} -->
<Button
    class={!tabsWithCategories.includes(id) || defaultFolders.includes(category.id) ? "" : $audioPlaylists[category.id] ? "context #playlist" : `context #category_${id}_button`}
    active={category.id === $drawerTabsData[id]?.[isSubmenu ? "activeSubmenu" : "activeSubTab"]}
    {red}
    on:click={(e) => {
        if (!editActive && !e.ctrlKey && !e.metaKey && !e.shiftKey && !e.target?.closest(".submenu_open")) setTab(category.id)
    }}
    bold={false}
    title={category.description ? category.description : category.path ? category.path : ""}
    style={isSubmenu && category.color ? `border-bottom: 2px solid ${category.color}` : ""}
>
    <span style="display: flex;align-items: center;width: calc(100% - 20px);">
        <Icon
            id={category.icon || "noIcon"}
            custom={["shows", "overlays", "templates"].includes(id) && ![undefined, "noIcon", "all", "variable", "trigger", "effects"].includes(category.icon)}
            select={["shows", "overlays", "templates"].includes(id) && !["all", "unlabeled", "favourites", "effects_library", "variables", "triggers", "effects"].includes(category.id)}
            selectData={{ id: "category_" + id, data: [category.id] }}
            size={isSubmenu ? 0.8 : 1}
            right
        />
        {#if isSubmenu}
            <span style="width: 100%;text-align: start;">
                <p style="margin: 3px;font-size: 0.9em;">
                    {#if category.name}
                        {category.name}
                    {:else}
                        <span style="opacity: 0.5;font-style: italic;pointer-events: none;"><T id="main.unnamed" /></span>
                    {/if}
                </p>
            </span>
        {:else}
            <span id={category.id} style="width: calc(100% - 15px);text-align: start;">
                {#if !tabsWithCategories.includes(id) || defaultFolders.includes(category.id)}
                    <p style="margin: 5px;"><T id={category.name} /></p>
                {:else}
                    <HiddenInput
                        value={category.default ? $dictionary[category.name.split(".")[0]]?.[category.name.split(".")[1]] : category.customName || category.name}
                        id={"category_" + id + "_" + category.id}
                        on:edit={(e) => changeName(e, category.id)}
                        bind:edit={editActive}
                    />
                {/if}
            </span>
        {/if}
    </span>

    {#if category.action && $actions[category.action]}
        <span style="padding: 0 5px;" title={$actions[category.action].name}>
            <Icon id={getActionIcon(category.action)} size={0.8} white />
        </span>
    {/if}

    {#if length[category.id]}
        <span style="opacity: 0.5;font-size: 0.9em;min-width: 15px;text-align: end;">
            {length[category.id]}
        </span>
    {/if}

    {#if !isSubmenu && category.submenu?.options?.length}
        <Button
            class="submenu_open"
            on:click={() => {
                drawerTabsData.update((a) => {
                    if (!a[id]) return a
                    let opened = a[id].openedSubmenus || []

                    let existingIndex = opened.findIndex((id) => id === category.id) ?? -1
                    if (existingIndex < 0) opened.push(category.id)
                    else {
                        opened.splice(existingIndex, 1)
                        delete a[id].activeSubmenu
                        if (category.openTrigger) category.openTrigger(category.id)
                    }

                    a[id].openedSubmenus = opened
                    return a
                })
            }}
            style="width: auto;position: absolute;right: 0;height: 100%;"
        >
            {#if $drawerTabsData[id]?.openedSubmenus?.includes(category.id)}
                <Icon class="submenu_open" id="arrow_down" size={1.4} style="fill: var(--secondary);" />
            {:else}
                <Icon class="submenu_open" id="arrow_right" size={1.4} style="fill: var(--text);" />
            {/if}
        </Button>
    {/if}
</Button>

<div class="submenus">
    {#if !isSubmenu && $drawerTabsData[id]?.openedSubmenus?.includes(category.id) && category.submenu?.options?.length}
        {#each category.submenu.options as option}
            <svelte:self {id} category={option} length={{ [option.id]: option.count }} categoryId={category.id} isSubmenu />
        {/each}
    {/if}
</div>

<style>
    .submenus {
        /* margin-left: 20px; */
        border-inline-start: 8px solid var(--primary-darker);
    }
</style>
