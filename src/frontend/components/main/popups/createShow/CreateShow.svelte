<script lang="ts">
    import { ShowObj } from "../../../../classes/Show"
    import { convertText, getQuickExample, trimNameFromString } from "../../../../converters/txt"
    import { activePopup, activeProject, activeShow, categories, dictionary, drawerTabsData, formatNewShow, quickTextCache, shows, special, splitLines } from "../../../../stores"
    import { newToast } from "../../../../utils/common"
    import { sortObject } from "../../../helpers/array"
    import { history } from "../../../helpers/history"
    import Icon from "../../../helpers/Icon.svelte"
    import { checkName } from "../../../helpers/show"
    import T from "../../../helpers/T.svelte"
    import Button from "../../../inputs/Button.svelte"
    import Checkbox from "../../../inputs/Checkbox.svelte"
    import CombinedInput from "../../../inputs/CombinedInput.svelte"
    import Dropdown from "../../../inputs/Dropdown.svelte"
    import NumberInput from "../../../inputs/NumberInput.svelte"
    import TextArea from "../../../inputs/TextArea.svelte"
    import TextInput from "../../../inputs/TextInput.svelte"
    import WebSearch from "./WebSearch.svelte"

    const isChecked = (e: any) => e.target.checked
    const changeValue = (e: any, key = "text") => {
        values[key] = e.target.value || ""

        // store text if popup is closed
        quickTextCache.set({ name: values.name, text: values.text })
    }
    let storedCache = $quickTextCache.text.length > 20 || $quickTextCache.fromSearch
    let values = {
        text: storedCache ? $quickTextCache.text : "",
        name: storedCache ? $quickTextCache.name : "",
        origin: ""
    }
    if ($quickTextCache.fromSearch) quickTextCache.set({ name: values.name, text: values.text })

    // CATEGORY

    const cats = [
        // { id: "", name: "—" }, // unlabeled
        ...sortObject(
            Object.keys($categories).map((key: string) => ({
                id: key,
                ...$categories[key]
            })),
            "name"
        ).map((cat) => ({
            id: cat.id,
            name: cat.default ? `$:${cat.name}:$` : cat.name
        }))
    ]

    let selectedCategory: any = cats[0]
    // get the selected category
    if ($drawerTabsData.shows?.activeSubTab && $categories[$drawerTabsData.shows.activeSubTab]) selectedCategory = cats.find((a) => a.id === $drawerTabsData.shows.activeSubTab)
    // get the category from the active show
    else if ($shows[$activeShow?.id || ""]?.category) selectedCategory = cats.find((a) => a.id === $shows[$activeShow?.id || ""]?.category)
    // set to "Songs" if it exists & nothing else if selected
    else if ($categories.song) selectedCategory = cats.find((a) => a.id === "song")
    // otherwise set to first category
    else if (cats.length > 1) selectedCategory = cats[1]

    // OPTIONS

    const createOptions = [
        { id: "text", name: "create_show.quick_lyrics", title: `${$dictionary.create_show?.quick_lyrics_tip} [Enter]`, icon: "text" },
        // { id: "clipboard", name: "clipboard", icon: "clipboard" },
        { id: "web", name: "create_show.web", title: `${$dictionary.create_show?.search_web} [Ctrl+F]`, icon: "search" },
        { id: "empty", name: "create_show.empty", title: `${$dictionary.new?.empty_show} [Ctrl+Enter]`, icon: "add" }
    ]
    let selectedOption = ""
    function selectOption(id: string) {
        if (id === "empty") {
            values.text = ""
            values.origin = ""
            textToShow()
        } else {
            selectedOption = id

            // look for existing shows with the same title
            if (values.name) {
                const exists = Object.values($shows).find((a) => a?.name?.toLowerCase() === values.name.toLowerCase())
                if (exists) newToast("$create_show.exists")
            }
        }
    }

    // WEB SEARCH

    function updateLyrics(e: any) {
        let data = e.detail || {}
        if (!data.lyrics) {
            selectedOption = ""
            return
        }

        // if name is lowercase, replace it
        if (data.title && values.name.toLowerCase() === values.name) {
            values.name = data.title
        }

        values.text = data.lyrics

        const metadata: string[] = []
        if (data.title) metadata.push(`Title=${data.title}`)
        if (data.artist) metadata.push(`Artist=${data.artist}`)
        if (metadata.length) values.text = `${metadata.join("\n")}\n\n${values.text}`

        if (data.source) values.origin = data.source.toLowerCase()
        selectedOption = "text"
    }

    // CREATE

    let showMore = false

    function textToShow() {
        let text = values.text
        if (typeof text !== "string") text = ""

        let sections = text.split("\n\n").filter((a) => a.length)

        // let metaData: string = ""
        // if (sections[1] && sections[0]?.split("\n").length < 3) metaData = sections.splice(0, 1)[0]
        let category = selectedCategory?.id?.length ? selectedCategory.id : null

        if (sections.length) {
            convertText({ name: values.name, category, text, origin: values.origin })
        } else {
            let show = new ShowObj(false, category)
            show.name = checkName(values.name)
            history({ id: "UPDATE", newData: { data: show, remember: { project: $activeProject } }, location: { page: "show", id: "show" } })
        }

        values = { name: "", text: "", origin: "" }
        quickTextCache.set({ name: "", text: "" })
        activePopup.set(null)
    }

    // SHORTCUTS

    function keydown(e: KeyboardEvent) {
        const ctrl = e.ctrlKey || e.metaKey
        if (e.key === "f" && ctrl) {
            e.preventDefault()
            selectOption("web")

            return
        }

        if (e.key === "Enter") {
            if (e.target?.closest(".edit") && !document.activeElement?.closest("#name")) return

            e.preventDefault()

            if (!ctrl) {
                selectOption("text")
                return
            }

            ;(document.activeElement as any)?.blur()
            textToShow()
        }
    }

    function getName(values) {
        if (values.name) return values.name
        if (values.text.trim().length) return trimNameFromString(values.text)
        return $dictionary.main?.unnamed
    }
</script>

<svelte:window on:keydown={keydown} />

{#if !selectedOption}
    <CombinedInput textWidth={30}>
        <p><T id="show.name" /></p>
        <TextInput id="name" autofocus value={values.name} on:input={(e) => changeValue(e, "name")} style="height: 30px;" />
    </CombinedInput>

    <CombinedInput textWidth={30}>
        <p><T id="show.category" /></p>
        <Dropdown options={cats} value={selectedCategory?.name} on:click={(e) => (selectedCategory = e.detail)} />
    </CombinedInput>

    <div class="choose">
        {#each createOptions as type, i}
            <Button title={type.title} disabled={type.id === "web" && !values.name?.trim()} on:click={() => selectOption(type.id)} style={i === 0 ? "border: 2px solid var(--focus);" : ""}>
                <Icon id={type.icon} size={4} white={type.id !== "text" || !values.text.length} />
                <p><T id={type.name} /></p>
            </Button>
        {/each}
    </div>
{:else}
    <Button class="popup-back" title={$dictionary.actions?.back} on:click={() => (selectedOption = "")}>
        <Icon id="back" size={2} white />
    </Button>
{/if}

{#if selectedOption === "text"}
    <div class="header"><T id="create_show.quick_lyrics" /></div>
    <!-- WIP buttons for paste / format(remove chords, remove empty lines), etc. -->
    <TextArea placeholder={getQuickExample()} style="height: 250px;min-width: 500px;" value={values.text} autofocus={!values.text} on:input={(e) => changeValue(e)} />
{:else if selectedOption === "web"}
    <WebSearch query={values.name} on:update={updateLyrics} />
{/if}

{#if selectedOption === "text"}
    <div class="create" style="margin-top: 10px;">
        {#if showMore}
            <CombinedInput>
                <p><T id="create_show.auto_groups" /></p>
                <div class="alignRight">
                    <Checkbox checked={$special.autoGroups !== false} on:change={(e) => special.set({ ...$special, autoGroups: isChecked(e) })} />
                </div>
            </CombinedInput>
            <CombinedInput title={$dictionary.create_show?.format_new_show_tip}>
                <p><T id="create_show.format_new_show" /></p>
                <div class="alignRight">
                    <Checkbox checked={$formatNewShow} on:change={(e) => formatNewShow.set(isChecked(e))} />
                </div>
            </CombinedInput>
            <CombinedInput title={$dictionary.create_show?.split_lines_tip}>
                <p><T id="create_show.split_lines" /></p>
                <NumberInput
                    value={$splitLines}
                    max={100}
                    on:change={(e) => {
                        splitLines.set(Number(e.detail))
                    }}
                />
            </CombinedInput>
        {:else}
            <CombinedInput>
                <Button on:click={() => (showMore = !showMore)} style="width: 100%;" dark center>
                    <div class="text" style="display: flex;align-items: center;padding: 0;">
                        <!-- settings -->
                        <Icon id="options" right white={!$formatNewShow && $special.autoGroups === false} />
                        <T id="edit.options" />

                        {#if Number($splitLines)}
                            <span class="name" style="font-size: 0.8em;">({$splitLines})</span>
                        {/if}
                    </div>
                </Button>
            </CombinedInput>
        {/if}

        <CombinedInput>
            <Button title="{$dictionary.timer?.create} [Ctrl+Enter]" disabled={values.text.trim().length === 0} on:click={textToShow} style="width: 100%;" dark center data-testid="create.show.popup.new.show">
                <div class="text" style="display: flex;align-items: center;padding: 0;">
                    {#if values.text.trim().length > 0}
                        <!-- showIcon -->
                        <Icon id="add" right />
                        <T id="timer.create" />
                        <!-- <T id="new.show" /> -->
                    {:else}
                        <Icon id="add" right white />
                        <T id="timer.create" />
                        <!-- <T id="create_show.empty" /> -->
                        <!-- <T id="new.empty_show" /> -->
                    {/if}

                    <span class="name" style="border: none;">{getName(values)}</span>
                    <!-- <span class="name">({#if values.name}{values.name}{:else}<T id="main.unnamed" />{/if})</span> -->
                </div>
            </Button>
        </CombinedInput>
    </div>
{/if}

<style>
    .choose {
        margin-top: 20px;

        width: 100%;
        display: flex;
        align-self: center;
        justify-content: space-between;
        gap: 10px;
    }

    .choose :global(button) {
        width: 180px;
        height: 180px;

        display: flex;
        gap: 10px;
        flex-direction: column;
        justify-content: center;
        flex: 1;

        /* border-radius: 3px; */
    }
    .choose p {
        display: flex;
        align-items: center;
    }

    .header {
        text-align: center;
        font-size: 0.9em;
        padding: 5px 0;
        background: var(--primary-darkest);
        font-weight: 600;
        opacity: 0.9;
    }

    .name {
        opacity: 0.5;
        margin-inline-start: 15px;
        align-content: center;
    }
</style>
