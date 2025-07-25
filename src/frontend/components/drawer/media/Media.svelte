<script lang="ts">
    import VirtualList from "@sveltejs/svelte-virtual-list"
    import { onDestroy } from "svelte"
    import { slide } from "svelte/transition"
    import { Main } from "../../../../types/IPC/Main"
    import { destroyMain, receiveMain, sendMain } from "../../../IPC/main"
    import { debounce } from "@tanstack/pacer"
    import {
        activeEdit,
        activeFocus,
        activeMediaTagFilter,
        activePopup,
        activeShow,
        dictionary,
        drawerTabsData,
        focusMode,
        labelsDisabled,
        media,
        mediaFolders,
        mediaOptions,
        outLocked,
        outputs,
        popupData,
        selectAllMedia,
        selected,
        sorted
    } from "../../../stores"
    import Icon from "../../helpers/Icon.svelte"
    import T from "../../helpers/T.svelte"
    import { clone, sortByName, sortFilenames } from "../../helpers/array"
    import { splitPath } from "../../helpers/get"
    import { getExtension, getFileName, getMediaType, isMediaExtension, removeExtension } from "../../helpers/media"
    import { getActiveOutputs, setOutput } from "../../helpers/output"
    import Button from "../../inputs/Button.svelte"
    import { clearBackground } from "../../output/clear"
    import Center from "../../system/Center.svelte"
    import BMDStreams from "../live/BMDStreams.svelte"
    import Cameras from "../live/Cameras.svelte"
    import NDIStreams from "../live/NDIStreams.svelte"
    import Screens from "../live/Screens.svelte"
    import Windows from "../live/Windows.svelte"
    import PlayerVideos from "../player/PlayerVideos.svelte"
    import Folder from "./Folder.svelte"
    import Media from "./MediaCard.svelte"
    import MediaGrid from "./MediaGrid.svelte"
    import { loadFromPixabay } from "./pixabay"
    import { loadFromUnsplash } from "./unsplash"

    export let active: string | null
    export let searchValue = ""
    export let streams: MediaStream[] = []

    type File = { path: string; favourite: boolean; name: string; extension: string; audio: boolean; folder?: boolean; stat?: any }
    let files: File[] = []

    let specialTabs = ["online", "screens", "cameras"]
    let notFolders = ["all", ...specialTabs]
    $: rootPath = notFolders.includes(active || "") ? "" : active !== null ? $mediaFolders[active]?.path || "" : ""
    $: path = notFolders.includes(active || "") ? "" : rootPath

    $: folderName = active === "all" ? "category.all" : active === "favourites" ? "category.favourites" : rootPath === path ? (active !== null ? $mediaFolders[active]?.name || "" : "") : splitPath(path).name

    async function loadFilesAsync() {
        fullFilteredFiles = []
        if ((onlineTab !== "pixabay" && onlineTab !== "unsplash") || activeView === "folder") return

        if (onlineTab === "pixabay") {
            fullFilteredFiles = await loadFromPixabay(searchValue || "landscape", activeView === "video")
        } else if (onlineTab === "unsplash") {
            fullFilteredFiles = await loadFromUnsplash(searchValue || "landscape")
        }
        loadAllFiles(fullFilteredFiles)
    }

    function setSubSubTab(id: string) {
        if (!active) return

        drawerTabsData.update((a) => {
            if (!a.media) a.media = { enabled: true, activeSubTab: active }
            if (!a.media.openedSubSubTab) a.media.openedSubSubTab = {}
            a.media.openedSubSubTab[active] = id
            return a
        })

        if (active === "screens") screenTab = id
        else if (active === "online") onlineTab = id
    }

    let screenTab = $drawerTabsData.media?.openedSubSubTab?.screens || "screens"
    let onlineTab = $drawerTabsData.media?.openedSubSubTab?.online || "youtube"

    $: if (active === "online" && onlineTab === "pixabay" && (searchValue !== null || activeView)) loadFilesAsync()
    $: if (active === "online" && onlineTab === "unsplash" && (searchValue !== null || activeView)) loadFilesAsync()

    // get list of files & folders
    let prevActive: null | string = null
    $: {
        if (prevActive === "online") activeView = "all"

        if (active === "online") {
            // WIP this resets on zoom
            activeView = "image"

            prevActive = active
        } else if (active === "favourites") {
            prevActive = active
            files = sortByName(
                Object.entries($media)
                    .map(([path, a]) => {
                        let p = splitPath(path)
                        let name = p.name
                        return { path, favourite: a.favourite === true, name, extension: p.extension, audio: a.audio === true }
                    })
                    .filter((a) => a.favourite === true && a.audio !== true)
            )

            filterFiles()
        } else if (active === "all") {
            if (active !== prevActive) {
                prevActive = active
                files = []
                fullFilteredFiles = []
                Object.values($mediaFolders).forEach((data) => sendMain(Main.READ_FOLDER, { path: data.path!, disableThumbnails: $mediaOptions.mode === "list" }))
            }
        } else if (path?.length) {
            if (path !== prevActive) {
                prevActive = path
                files = []
                fullFilteredFiles = []
                sendMain(Main.READ_FOLDER, { path, listFilesInFolders: true, disableThumbnails: $mediaOptions.mode === "list" })
            }
        } else {
            // screens && cameras
            prevActive = active
        }
    }

    let filesInFolders: File[] = []
    let folderFiles: { [key: string]: string[] } = {}

    let listenerId = receiveMain(Main.READ_FOLDER, (data) => {
        filesInFolders = sortFilenames(data.filesInFolders || [])

        if (active !== "all" && data.path !== path) return

        files.push(...(data.files.filter((file) => file.folder || isMediaExtension(file.extension)) as any))
        files = sortFilenames(files).sort((a, b) => (a.folder === b.folder ? 0 : a.folder ? -1 : 1))

        files = files.map((a) => ({ ...a, path: a.folder ? a.path : a.path }))

        // set valid files in folder
        folderFiles = {}
        Object.keys(data.folderFiles).forEach((path) => {
            folderFiles[path] = data.folderFiles[path].filter((file) => file.folder || isMediaExtension(file.extension))
        })

        filterFiles()
    })
    onDestroy(() => destroyMain(listenerId))

    let scrollElem: HTMLElement | undefined

    // arrow selector
    let activeFile: null | number = null
    let allFiles: string[] = []
    let content = allFiles.length

    $: showUpdate($activeShow)
    function showUpdate(a) {
        if (a?.type !== "video" && a?.type !== "image") activeFile = null
    }

    // filter files
    let activeView = "all" // keyof typeof nextActiveView
    let filteredFiles: File[] = []
    $: if (activeView || $activeMediaTagFilter) filterFiles()
    $: if (searchValue !== undefined) filterSearch()

    function filterFiles() {
        if (active === "online" || active === "screens" || active === "cameras") return

        // filter files
        if (activeView === "all") filteredFiles = files.filter((a) => active !== "all" || !a.folder)
        else filteredFiles = files.filter((a) => (activeView === "folder" && active !== "all" && a.folder) || (!a.folder && activeView === getMediaType(a.extension)))

        // filter by tag
        if ($activeMediaTagFilter.length) {
            filteredFiles = filteredFiles.filter((a) => !a.folder && $media[a.path]?.tags?.length && !$activeMediaTagFilter.find((tagId) => !$media[a.path].tags!.includes(tagId)))
        }

        // remove folders with no content
        filteredFiles = filteredFiles.filter((a) => !a.folder || !folderFiles[a.path] || folderFiles[a.path].length > 0)

        // reset arrow selector
        loadAllFiles(filteredFiles)

        filterSearch()

        // scroll to top
        scrollElem?.scrollTo(0, 0)
    }

    function loadAllFiles(f: File[]) {
        allFiles = [...f.filter((a) => !a.folder).map((a) => a.path)]
        if ($activeShow !== null && allFiles.includes($activeShow.id)) activeFile = allFiles.findIndex((a) => a === $activeShow!.id)
        else activeFile = null
        content = allFiles.length
    }

    // search
    const filter = (s: string) => s.toLowerCase().replace(/[.,\/#!?$%\^&\*;:{}=\-_`~() ]/g, "")
    let fullFilteredFiles: File[] = []
    function filterSearch() {
        fullFilteredFiles = clone(filteredFiles)
        if (searchValue.length > 1) fullFilteredFiles = [...fullFilteredFiles, ...filesInFolders].filter((a) => filter(a.name).includes(filter(searchValue)))

        // scroll to top
        document.querySelector("svelte-virtual-list-viewport")?.scrollTo(0, 0)
    }

    let sortedFiles: File[] = []
    $: if (fullFilteredFiles && $sorted) sortFiles()
    function sortFiles() {
        let type = $sorted.media?.type || "name"

        let files = clone(fullFilteredFiles)

        if (searchValue.length > 1 || type === "name") files = files
        else if (type === "name_des") files = files.reverse()
        else if (type === "created") files = files.sort((a, b) => b.stat?.birthtimeMs - a.stat?.birthtimeMs)
        else if (type === "modified") files = files.sort((a, b) => b.stat?.mtimeMs - a.stat?.mtimeMs)

        sortedFiles = files.sort((a, b) => (a.folder === b.folder ? 0 : a.folder ? -1 : 1))
    }

    const debouncedColumnUpdate = debounce((deltaY: number) => {
        mediaOptions.set({ ...$mediaOptions, columns: Math.max(2, Math.min(10, $mediaOptions.columns + (deltaY < 0 ? -100 : 100) / 100)) })
    }, { wait: 50 }) // 50ms debounce for responsive column adjustments

    function wheel(e: any) {
        if (!e.ctrlKey && !e.metaKey) return
        
        e.preventDefault()
        debouncedColumnUpdate(e.deltaY)
    }

    const shortcuts = {
        ArrowRight: () => {
            if ($activeEdit.items.length) return
            if (activeFile === null || activeFile < content - 1) activeFile = activeFile === null ? 0 : activeFile + 1
        },
        ArrowLeft: () => {
            if ($activeEdit.items.length) return
            if (activeFile === null || activeFile > 0) activeFile = activeFile === null ? content - 1 : activeFile - 1
        },
        Backspace: () => {
            if (rootPath === path) return
            goBack()
        }
    }

    $: if (activeFile !== null) selectMedia()
    function selectMedia() {
        if (activeFile === null) return

        let path = allFiles[activeFile]
        if (!path) return

        activeEdit.set({ id: path, type: "media", items: [] })
        let name = removeExtension(getFileName(path))
        let type = getMediaType(getExtension(path))

        if ($focusMode) activeFocus.set({ id: path, type })
        else activeShow.set({ id: path, name, type })
    }

    function keydown(e: KeyboardEvent) {
        if (e.key === "Enter" && searchValue.length > 1 && e.target?.closest(".search")) {
            if (fullFilteredFiles.length) {
                let file = fullFilteredFiles[0]

                if ($focusMode) activeFocus.set({ id: file.path, type: getMediaType(file.extension) })
                else activeShow.set({ id: file.path, name: file.name, type: getMediaType(file.extension) })

                activeFile = filteredFiles.findIndex((a) => a.path === file.path)
                if (activeFile < 0) activeFile = null
            }
        }

        if (e.target?.closest("input") || e.target?.closest(".edit") || !allFiles.length) return

        if ((e.ctrlKey || e.metaKey) && shortcuts[e.key]) {
            // e.preventDefault()
            shortcuts[e.key]()
        }
    }

    function goBack() {
        const lastSlash = path.lastIndexOf("\\") > -1 ? path.lastIndexOf("\\") : path.lastIndexOf("/")
        const folder = path.slice(0, lastSlash)
        path = folder.length > rootPath.length ? folder : rootPath
    }

    const slidesViews: any = { grid: "list", list: "grid" }
    const nextActiveView = { all: "folder", folder: "image", image: "video", video: "all" }
    $: if (notFolders.includes(active || "") && activeView === "folder") activeView = "image"

    let zoomOpened = false
    function mousedown(e: any) {
        if (e.target.closest(".zoom_container") || e.target.closest("button")) return

        zoomOpened = false
    }

    $: currentOutput = $outputs[getActiveOutputs()[0]] || {}

    // select all
    $: if ($selectAllMedia) selectAll()
    function selectAll() {
        let data = sortedFiles
            .filter((a) => a.extension)
            .map((file) => {
                let type = getMediaType(file.extension)
                let name = file.name.slice(0, file.name.lastIndexOf("."))
                return { name, path: file.path, type }
            })

        selected.set({ id: "media", data })
        selectAllMedia.set(false)
    }
</script>

<!-- TODO: download pixabay images!!! -->
<!-- TODO: pexels images ? -->

<svelte:window on:keydown={keydown} on:mousedown={mousedown} />

{#if active === "screens"}
    <div class="tabs">
        <Button style="flex: 1;" active={screenTab === "screens"} on:click={() => setSubSubTab("screens")} center>
            <Icon size={1.2} id="screen" right />
            <p><T id="live.screens" /></p>
        </Button>
        <Button style="flex: 1;" active={screenTab === "windows"} on:click={() => setSubSubTab("windows")} center>
            <Icon size={1.2} id="window" right />
            <p><T id="live.windows" /></p>
        </Button>
        <Button style="flex: 1;" active={screenTab === "ndi"} on:click={() => setSubSubTab("ndi")} center>
            <Icon size={1.2} id="ndi" right />
            <p>NDI</p>
        </Button>
        <!-- BLACKMAGIC CURRENTLY NOT WORKING -->
        <!-- <Button style="flex: 1;" active={screenTab === "blackmagic"} on:click={() => setSubSubTab("blackmagic")} center>
            <Icon size={1.2} id="blackmagic" right />
            <p>Blackmagic</p>
        </Button> -->
    </div>
{:else if active === "online"}
    <div class="tabs">
        <Button style="flex: 1;" active={onlineTab === "youtube"} on:click={() => setSubSubTab("youtube")} center>
            <Icon style="fill: {onlineTab !== 'youtube' ? 'white' : '#ff0000'};" size={1.2} id="youtube" right />
            <p>YouTube</p>
        </Button>
        <Button style="flex: 1;" active={onlineTab === "vimeo"} on:click={() => setSubSubTab("vimeo")} center>
            <Icon style="fill: {onlineTab !== 'vimeo' ? 'white' : '#17d5ff'};" size={1.2} id="vimeo" right />
            <p>Vimeo</p>
        </Button>
        <Button style="flex: 1;" active={onlineTab === "pixabay"} on:click={() => setSubSubTab("pixabay")} center>
            <Icon style="fill: {onlineTab !== 'pixabay' ? 'white' : '#00ab6b'};" size={1.2} id="pixabay" box={48} right />
            <p>Pixabay</p>
        </Button>
        <Button style="flex: 1;" active={onlineTab === "unsplash"} on:click={() => setSubSubTab("unsplash")} center>
            <!-- #111111 -->
            <Icon style="fill: {onlineTab !== 'unsplash' ? 'white' : '#bbbbbb'};" size={1.2} id="unsplash" right />
            <p>Unsplash</p>
        </Button>
    </div>
{/if}

<!-- MAIN -->

<div class="scroll" style="flex: 1;overflow-y: auto;" bind:this={scrollElem} on:wheel|passive={wheel}>
    <div class="grid" class:list={$mediaOptions.mode === "list"} style="height: 100%;">
        {#if active === "online" && (onlineTab === "youtube" || onlineTab === "vimeo")}
            <div class="gridgap">
                <PlayerVideos active={onlineTab} {searchValue} />
            </div>
        {:else if active === "screens"}
            <div class="gridgap">
                {#if screenTab === "screens"}
                    <Screens bind:streams />
                {:else if screenTab === "ndi"}
                    <NDIStreams />
                {:else if screenTab === "blackmagic"}
                    <BMDStreams />
                {:else}
                    <Windows bind:streams {searchValue} />
                {/if}
            </div>
        {:else if active === "cameras"}
            <div class="gridgap">
                <Cameras
                    on:click={({ detail }) => {
                        let e = detail.event
                        let cam = detail.cam

                        if ($outLocked || e.ctrlKey || e.metaKey) return
                        if (currentOutput.out?.background?.id === cam.id) clearBackground()
                        else setOutput("background", { name: cam.name, id: cam.id, cameraGroup: cam.cameraGroup, type: "camera" })
                    }}
                />
            </div>
        {:else if sortedFiles.length}
            <div class="context #media" style="display: contents;">
                {#key sortedFiles}
                    {#if $mediaOptions.mode === "grid"}
                        <MediaGrid items={sortedFiles} columns={$mediaOptions.columns} let:item>
                            {#if item.folder}
                                <Folder bind:rootPath={path} name={item.name} path={item.path} mode={$mediaOptions.mode} folderPreview={sortedFiles.length < 20} />
                            {:else}
                                <Media
                                    credits={item.credits || {}}
                                    name={item.name || ""}
                                    path={item.path}
                                    thumbnailPath={item.previewUrl || ($mediaOptions.columns < 3 ? "" : item.thumbnailPath)}
                                    type={getMediaType(item.extension)}
                                    shiftRange={sortedFiles.map((a) => ({ ...a, type: getMediaType(a.extension), name: removeExtension(a.name) }))}
                                    bind:activeFile
                                    {allFiles}
                                    {active}
                                />
                            {/if}
                        </MediaGrid>
                    {:else}
                        <VirtualList items={sortedFiles} let:item={file}>
                            {#if file.folder}
                                <Folder bind:rootPath={path} name={file.name} path={file.path} mode={$mediaOptions.mode} />
                            {:else}
                                <Media
                                    credits={file.credits || {}}
                                    thumbnail={$mediaOptions.mode !== "list"}
                                    name={file.name || ""}
                                    path={file.path}
                                    type={getMediaType(file.extension)}
                                    shiftRange={sortedFiles.map((a) => ({ ...a, type: getMediaType(a.extension), name: removeExtension(a.name) }))}
                                    bind:activeFile
                                    {allFiles}
                                    {active}
                                />
                            {/if}
                        </VirtualList>
                    {/if}
                {/key}
            </div>
        {:else}
            <div class={specialTabs.includes(active || "") ? "" : "context #media"} style="display: contents;">
                <Center style="opacity: 0.2;">
                    <Icon id="noImage" size={5} white />
                </Center>
            </div>
        {/if}
    </div>
</div>

{#if active !== "cameras"}
    <div class="tabs">
        {#if active === "screens" || active === "online"}
            <span style="flex: 1;"></span>
        {:else}
            {#if active !== "all" && active !== "favourites"}
                <Button disabled={rootPath === path} title={$dictionary.actions?.back} on:click={goBack}>
                    <Icon size={1.3} id="back" />
                </Button>
            {/if}
            <!-- <Button disabled={rootPath === path} title={$dictionary.actions?.home} on:click={() => (path = rootPath)}>
            <Icon size={1.3} id="home" />
        </Button> -->
            <span style="flex: 1;display: flex;align-items: center;justify-content: center;">
                {#key folderName}
                    {#if folderName.includes(".")}
                        <T id={folderName} />
                    {:else}
                        {folderName}
                    {/if}
                {/key}

                {#if content}
                    <span style="opacity: 0.6;font-size: 0.8em;margin-inline-start: 5px;">({content})</span>
                {/if}
            </span>

            <!-- <div class="seperator" />

            <Button disabled={!allFiles.length || activeFile === 0} on:click={() => (activeFile = activeFile === null ? content - 1 : activeFile - 1)}>
                <Icon size={1.3} id="previous" />
            </Button>
            <p style="opacity: 0.8;">{activeFile === null ? "" : activeFile + 1 + "/"}{content}</p>
            <Button disabled={!allFiles.length || activeFile === content - 1} on:click={() => (activeFile = activeFile === null ? 0 : activeFile + 1)}>
                <Icon size={1.3} id="next" />
            </Button> -->
        {/if}

        {#if active !== "screens"}
            {#if active === "online" && (onlineTab === "youtube" || onlineTab === "vimeo")}
                <Button
                    style="width: 100%;"
                    on:click={() => {
                        popupData.set({ active: onlineTab })
                        activePopup.set("player")
                    }}
                    center
                >
                    <Icon id="add" right={!$labelsDisabled} />
                    {#if !$labelsDisabled}<T id="settings.add" />{/if}
                </Button>
            {:else}
                <div class="seperator" />

                {#if active === "online" && onlineTab === "unsplash"}
                    <!-- only images!! -->
                {:else if active === "online" && onlineTab === "pixabay"}
                    <Button title={$dictionary.media?.image} on:click={() => (activeView = "image")}>
                        <Icon size={1.2} id="image" white={activeView !== "image"} />
                    </Button>
                    <Button title={$dictionary.media?.video} on:click={() => (activeView = "video")}>
                        <Icon size={1.2} id="video" white={activeView !== "video"} />
                    </Button>

                    <div class="seperator" />
                {:else}
                    <Button title={$dictionary.media?.[activeView]} on:click={() => (activeView = nextActiveView[activeView])}>
                        <Icon size={1.2} id={activeView === "all" ? "media" : activeView} white={activeView === "all"} />
                    </Button>

                    <div class="seperator" />
                {/if}

                <Button
                    on:click={() =>
                        mediaOptions.update((a) => {
                            a.mode = slidesViews[$mediaOptions.mode]
                            return a
                        })}
                    title={$dictionary.show?.[$mediaOptions.mode]}
                >
                    <Icon size={1.3} id={$mediaOptions.mode} white />
                </Button>

                {#if active !== "online"}
                    <Button on:click={() => (zoomOpened = !zoomOpened)} disabled={$mediaOptions.mode === "list"} title={$dictionary.actions?.zoom}>
                        <Icon size={1.3} id="zoomIn" white />
                    </Button>
                    {#if zoomOpened}
                        <div class="zoom_container" transition:slide={{ duration: 150 }}>
                            <Button style="padding: 0 !important;width: 100%;" on:click={() => mediaOptions.set({ ...$mediaOptions, columns: 5 })} bold={false} center>
                                <p class="text" title={$dictionary.actions?.resetZoom}>{(100 / $mediaOptions.columns).toFixed()}%</p>
                            </Button>
                            <Button disabled={$mediaOptions.columns <= 2} on:click={() => mediaOptions.set({ ...$mediaOptions, columns: Math.max(2, $mediaOptions.columns - 1) })} title={$dictionary.actions?.zoomIn} center>
                                <Icon size={1.3} id="add" white />
                            </Button>
                            <Button disabled={$mediaOptions.columns >= 10} on:click={() => mediaOptions.set({ ...$mediaOptions, columns: Math.min(10, $mediaOptions.columns + 1) })} title={$dictionary.actions?.zoomOut} center>
                                <Icon size={1.3} id="remove" white />
                            </Button>
                        </div>
                    {/if}
                {/if}
            {/if}
        {/if}
    </div>
{/if}

<style>
    .tabs {
        display: flex;
        position: relative;
        background-color: var(--primary-darkest);
        align-items: center;
    }

    .grid {
        display: flex;
        flex-wrap: wrap;
        flex: 1;
        place-content: flex-start;
    }

    .grid :global(.selectElem) {
        outline-offset: -3px;
    }
    /* .grid :global(#media.isSelected .main) {
        z-index: -1;
    } */

    .grid :global(svelte-virtual-list-viewport) {
        width: 100%;
        padding: 5px;
    }

    .gridgap {
        display: flex;
        flex-wrap: wrap;
        align-content: flex-start;
        padding: 5px;

        width: 100%;
        height: 100%;

        overflow-y: auto;
        overflow-x: hidden;
    }

    .text {
        opacity: 0.8;
        text-align: center;
        padding: 0.5em 0;
    }

    .seperator {
        width: 1px;
        height: 100%;
        background-color: var(--primary);
    }

    .zoom_container {
        position: absolute;
        inset-inline-end: 0;
        top: 0;
        transform: translateY(-100%);
        overflow: hidden;

        flex-direction: column;
        width: auto;
        background-color: inherit;
    }
</style>
