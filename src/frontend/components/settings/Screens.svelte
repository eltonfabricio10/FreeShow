<script lang="ts">
    import { onMount } from "svelte"
    import { OUTPUT } from "../../../types/Channels"
    import { Main } from "../../../types/IPC/Main"
    import { requestMain } from "../../IPC/main"
    import { activePopup, alertMessage, currentOutputSettings, dictionary, outputDisplay, outputs, styles } from "../../stores"
    import { triggerClickOnEnterSpace } from "../../utils/clickable"
    import { send } from "../../utils/request"
    import { clone, keysToID, sortByName } from "../helpers/array"
    import Icon from "../helpers/Icon.svelte"
    import T from "../helpers/T.svelte"
    import Button from "../inputs/Button.svelte"
    import Checkbox from "../inputs/Checkbox.svelte"
    import CombinedInput from "../inputs/CombinedInput.svelte"
    import NumberInput from "../inputs/NumberInput.svelte"

    export let activateOutput = false

    let options: any[] = []
    $: options = sortByName(keysToID($outputs))
    $: if (options.length && (!$currentOutputSettings || !$outputs[$currentOutputSettings])) currentOutputSettings.set(options[0].id)

    let screens: any[] = []

    let screenId: string | null = null
    $: screenId = $currentOutputSettings
    let currentScreen: any = {}
    $: currentScreen = screenId ? $outputs[screenId] || {} : {}

    // find matching screen
    $: if (!currentScreen.screen && screens.length) {
        let match = screens.find((screen) => JSON.stringify(screen.bounds) === JSON.stringify(currentScreen.bounds))
        if (match?.id) {
            outputs.update((a: any) => {
                a[screenId!].screen = match.id.toString()
                return a
            })
        }
    }

    let minPosX: number | null = null
    let minPosY: number | null = null
    let totalScreensWidth = 0
    let totalScreensHeight = 0

    onMount(async () => {
        const displays = await requestMain(Main.GET_DISPLAYS)
        let sortedScreens = displays.sort(sortScreensByPosition)
        screens = sortedScreens.sort(internalFirst)

        // get min/max bounds
        minPosX = null
        minPosY = null
        let maxPosX: null | number = null
        let maxPosY: null | number = null
        clone(screens).forEach(({ bounds }) => {
            if (minPosX === null || bounds.x < minPosX) minPosX = bounds.x
            if (minPosY === null || bounds.y < minPosY) minPosY = bounds.y
            if (maxPosX === null || bounds.x + bounds.width > maxPosX) maxPosX = bounds.x + bounds.width
            if (maxPosY === null || bounds.y + bounds.height > maxPosY) maxPosY = bounds.y + bounds.height
        })

        totalScreensWidth = maxPosX !== null && minPosX !== null ? (maxPosX - minPosX) / 2 : 0
        totalScreensHeight = maxPosY !== null && minPosY !== null ? (maxPosY - minPosY) / 2 : 0

        // don't overlap buttons (& allow scroll bars to work)
        totalScreensWidth = Math.min(totalScreensWidth, 4800)
        totalScreensHeight = Math.min(totalScreensHeight, 1000)

        // make all values start at 0
        screens.forEach((a, i) => {
            screens[i].previewBounds = {
                x: a.bounds.x - (minPosX || 0),
                y: a.bounds.y - (minPosY || 0)
            }
        })
    })

    // const fakeScreen0 = {
    //     bounds: { x: -864 - 1536, y: -452, width: 1536, height: 1536 },
    //     id: 2528732444,
    //     internal: false,
    // }
    // const fakeScreen = {
    //     bounds: { x: -864, y: -452, width: 864, height: 1536 },
    //     id: 2528732444,
    //     internal: false,
    // }
    // const fakeScreen2 = {
    //     bounds: { x: 1920 + 1536, y: 0, width: 1536, height: 1536 },
    //     id: 2528732444,
    //     internal: false,
    // }
    // const fakeScreen3 = {
    //     bounds: { x: 1920 + 1536 + 1536, y: 0, width: 1536, height: 1536 },
    //     id: 2528732444,
    //     internal: false,
    // }

    function internalFirst(a, b) {
        return b.internal - a.internal
    }
    function sortScreensByPosition(a, b) {
        let aX = a.bounds.x
        let bX = b.bounds.x

        return aX - bX
    }

    function changeOutputScreen(e: any) {
        if (!currentScreen || !screenId) return

        let alreadySelected = $outputs[screenId]?.screen === e.detail.id.toString()

        let bounds = e.detail.bounds
        let keyOutput = currentScreen.keyOutput
        outputs.update((a) => {
            if (!a[screenId]) return a

            a[screenId].bounds = { x: bounds.x, y: bounds.y, width: bounds.width, height: bounds.height }
            a[screenId].screen = e.detail.id.toString()
            // a[screenId].kiosk = true

            if (keyOutput && a[keyOutput]) {
                a[keyOutput].bounds = { x: bounds.x, y: bounds.y, width: bounds.width, height: bounds.height }
                a[keyOutput].screen = e.detail.id.toString()
            }
            return a
        })

        setTimeout(() => {
            let enabled = activateOutput || $outputDisplay
            send(OUTPUT, ["DISPLAY"], { enabled, output: { id: screenId, ...currentScreen }, reset: true })
            // send(OUTPUT, ["TOGGLE_KIOSK"], { id: screenId, enabled: true })
            // setTimeout(() => {
            send(OUTPUT, ["UPDATE_BOUNDS"], { id: screenId, ...currentScreen })
            // }, 100)

            if (keyOutput) {
                send(OUTPUT, ["DISPLAY"], { enabled, output: { id: keyOutput, ...currentScreen }, reset: true })
                send(OUTPUT, ["UPDATE_BOUNDS"], { id: keyOutput, ...currentScreen })
            }
        }, 100)

        if (activateOutput || alreadySelected) {
            activePopup.set(null)
            alertMessage.set("")
        }
    }

    function lockScreen() {
        if (!screenId) return

        outputs.update((a) => {
            if (!a[screenId]) return a

            a[screenId].boundsLocked = !a[screenId].boundsLocked
            return a
        })
    }

    function identifyScreens() {
        send(OUTPUT, ["IDENTIFY_SCREENS"], screens)
    }

    const isChecked = (e: any) => e.target.checked

    let editCropping = false
    const previewSize = 0.08
    // get any style cropping if not set on window
    $: cropping = clone((currentScreen.cropping === undefined && currentScreen.style ? $styles[currentScreen.style]?.cropping : currentScreen.cropping) || {})
    function updateCropping(value: string | number, side: string) {
        if (!screenId) return

        outputs.update((a) => {
            if (!a[screenId].cropping) {
                if (currentScreen.style && $styles[currentScreen.style]?.cropping) a[screenId].cropping = clone($styles[currentScreen.style].cropping)
                else a[screenId].cropping = { top: 0, right: 0, bottom: 0, left: 0 }
            }

            a[screenId].cropping![side] = Number(value)
            return a
        })
    }
    function getCroppedStyle(cropping) {
        let style = ""
        Object.keys(cropping).forEach((key) => {
            if (cropping[key]) style += `padding-${key}: ${cropping[key] * previewSize}px;`
        })
        return style
    }

    let editEdgeBlending = false
    $: blending = currentScreen.blending || {}
    function updateBlending(value: string | number, side: string) {
        if (!screenId) return

        outputs.update((a) => {
            if (!a[screenId].blending) a[screenId].blending = { left: 0, right: 0, rotate: 90, opacity: 50, centered: false, offset: 0 }
            a[screenId].blending![side] = Number(value)
            return a
        })
    }
    function getBlendingStyle(blending) {
        if (!blending.left && !blending.right) return ""

        const opacity = (blending.opacity ?? 50) / 100
        const center = 50 + Number(blending.offset || 0)
        if (blending.centered) return `-webkit-mask-image: linear-gradient(${blending.rotate ?? 90}deg, rgb(0, 0, 0) ${center - blending.left}%, rgba(0, 0, 0, ${opacity}) ${center}%, rgb(0, 0, 0) ${center + Number(blending.right)}%);`
        return `-webkit-mask-image: linear-gradient(${blending.rotate ?? 90}deg, rgba(0, 0, 0, ${opacity}) 0%, rgb(0, 0, 0) ${blending.left}%, rgb(0, 0, 0) ${100 - blending.right}%, rgba(0, 0, 0, ${opacity}) 100%);`
    }
</script>

{#if editCropping}
    <Button class="popup-back" title={$dictionary.actions?.back} on:click={() => (editCropping = false)}>
        <Icon id="back" size={2} white />
    </Button>

    <p class="tip"><T id="screen.cropping_tip" /></p>

    <CombinedInput>
        <p><T id="screen.top" /></p>
        <NumberInput value={cropping.top || 0} max={currentScreen.bounds?.height * 0.9 - (cropping.bottom || 0)} on:change={(e) => updateCropping(e.detail, "top")} />
    </CombinedInput>
    <CombinedInput>
        <p><T id="screen.right" /></p>
        <NumberInput value={cropping.right || 0} max={currentScreen.bounds?.width * 0.9 - (cropping.left || 0)} on:change={(e) => updateCropping(e.detail, "right")} />
    </CombinedInput>
    <CombinedInput>
        <p><T id="screen.bottom" /></p>
        <NumberInput value={cropping.bottom || 0} max={currentScreen.bounds?.height * 0.9 - (cropping.top || 0)} on:change={(e) => updateCropping(e.detail, "bottom")} />
    </CombinedInput>
    <CombinedInput>
        <p><T id="screen.left" /></p>
        <NumberInput value={cropping.left || 0} max={currentScreen.bounds?.width * 0.9 - (cropping.right || 0)} on:change={(e) => updateCropping(e.detail, "left")} />
    </CombinedInput>

    <!-- preview -->
    <div class="preview" style="margin-top: 20px;">
        <div class="border" style="width: {currentScreen.bounds?.width * previewSize}px;height: {currentScreen.bounds?.height * previewSize}px;">
            <div class="cropped" style={getCroppedStyle(cropping)}>
                <div
                    class="previewWindow"
                    style="aspect-ratio: {currentScreen.bounds?.width} / {currentScreen.bounds?.height};max-height: {(currentScreen.bounds?.height - (cropping.top || 0) - (cropping.bottom || 0)) * previewSize}px;max-width: {(currentScreen.bounds
                        ?.width -
                        (cropping.left || 0) -
                        (cropping.right || 0)) *
                        previewSize}px;"
                ></div>
            </div>
        </div>
    </div>
{:else if editEdgeBlending}
    <Button class="popup-back" title={$dictionary.actions?.back} on:click={() => (editEdgeBlending = false)}>
        <Icon id="back" size={2} white />
    </Button>

    <p class="tip"><T id="screen.edge_blending_tip" /></p>

    <CombinedInput>
        <p><T id="screen.left" /></p>
        <NumberInput value={blending.left || 0} max={500} on:change={(e) => updateBlending(e.detail, "left")} />
    </CombinedInput>
    <CombinedInput>
        <p><T id="screen.right" /></p>
        <NumberInput value={blending.right || 0} max={100} on:change={(e) => updateBlending(e.detail, "right")} />
    </CombinedInput>
    <CombinedInput>
        <p><T id="edit.rotation" /></p>
        <NumberInput value={blending.rotate ?? 90} max={360} on:change={(e) => updateBlending(e.detail, "rotate")} />
    </CombinedInput>
    <CombinedInput>
        <p><T id="edit.opacity" /></p>
        <NumberInput value={blending.opacity ?? 50} max={100} step={5} on:change={(e) => updateBlending(e.detail, "opacity")} />
    </CombinedInput>
    <CombinedInput>
        <p><T id="screen.centered" /></p>
        <div class="alignRight">
            <Checkbox checked={blending.centered} on:change={(e) => updateBlending(isChecked(e), "centered")} />
        </div>
    </CombinedInput>
    {#if blending.centered}
        <CombinedInput>
            <p><T id="edit.offset" /></p>
            <NumberInput value={blending.offset} min={-50} max={50} on:change={(e) => updateBlending(e.detail, "offset")} />
        </CombinedInput>
    {/if}

    <!-- preview -->
    <div class="preview" style="margin-top: 20px;">
        <div class="border" style="background-color: white;width: {currentScreen.bounds?.width * previewSize}px;height: {currentScreen.bounds?.height * previewSize}px;">
            <div class="cropped" style={getBlendingStyle(blending)}>
                <div class="previewWindow" style="aspect-ratio: {currentScreen.bounds?.width} / {currentScreen.bounds?.height};"></div>
            </div>
        </div>
    </div>
{:else}
    <p class="tip"><T id="settings.{currentScreen.boundsLocked ? 'output_locked' : 'select_display'}" /></p>
    <Button disabled={currentScreen.boundsLocked} on:click={() => activePopup.set("change_output_values")} title={activateOutput ? $dictionary.popup?.change_output_values : $dictionary.settings?.manual_input_hint} style="width: 100%;" dark center>
        <Icon id="options" right />
        {#if activateOutput}
            <p><T id="settings.manual_input_hint" /></p>
        {:else}
            <p><T id="popup.change_output_values" /></p>
        {/if}
    </Button>

    {#if !activateOutput}
        <div style="display: flex;">
            <Button on:click={() => (editCropping = true)} style="flex: 1;" dark center>
                <Icon id="resize" right />
                <p><T id="settings.cropping" /></p>
            </Button>

            <Button on:click={() => (editEdgeBlending = true)} style="flex: 1;" dark center>
                <Icon id="gradient" right />
                <p><T id="settings.edge_blending" /></p>
            </Button>
        </div>
    {/if}

    <br />

    <div class="content">
        {#if screens.length}
            <div class="screens" style="transform: translate(-{totalScreensWidth}px, -{totalScreensHeight}px)">
                <!-- {#if !currentScreen.screen || !screens.find((a) => a.id.toString() === currentScreen.screen)} -->
                <div
                    style="position: absolute;width: {currentScreen.bounds?.width}px;height: {currentScreen.bounds?.height}px;inset-inline-start: {currentScreen.bounds?.x - (minPosX ? minPosX : 0)}px;top: {currentScreen.bounds?.y -
                        (minPosY ? minPosY : 0)}px;"
                >
                    {#if currentScreen.screen}
                        <span style="z-index: 2;position: absolute;top: 50%;inset-inline-start: 50%;transform: translate(-50%, -50%);">{screens.findIndex((a) => JSON.stringify(currentScreen.bounds) === JSON.stringify(a.bounds)) + 1 || ""}</span>
                    {/if}
                    <!-- Current screen position -->
                    <div class="screen noClick" style="width: 100%;height: 100%;{currentScreen.screen && screens.find((a) => a.id.toString() === currentScreen.screen) ? 'opacity: 1;' : ''}"></div>
                    {#if !activateOutput}
                        <Button class="lock" on:click={lockScreen} red={currentScreen.boundsLocked} center>
                            <div style="display: contents;" title={currentScreen.boundsLocked ? $dictionary.preview?._unlock : $dictionary.preview?._lock}>
                                <Icon id={currentScreen.boundsLocked ? "locked" : "unlocked"} size={1.1} />
                            </div>
                        </Button>
                    {/if}
                </div>
                <!-- {/if} -->

                {#each screens as screen, i}
                    <!-- class:active={$outputs[screenId || ""]?.screen === screen.id.toString()} -->
                    <div
                        class="screen"
                        class:disabled={currentScreen?.forcedResolution || currentScreen.boundsLocked}
                        style="width: {screen.bounds.width}px;height: {screen.bounds.height}px;inset-inline-start: {screen.previewBounds.x}px;top: {screen.previewBounds.y}px;"
                        role="button"
                        tabindex="0"
                        on:click={() => {
                            if (currentScreen?.forcedResolution || currentScreen.boundsLocked) return

                            // WIP this will not always change correct output if multiple & "activateOutput"
                            changeOutputScreen({ detail: { id: screen.id, bounds: screen.bounds } })
                        }}
                        on:keydown={triggerClickOnEnterSpace}
                    >
                        {i + 1}
                    </div>
                {/each}
            </div>
        {:else}
            <T id="remote.loading" />
        {/if}
    </div>

    {#if !currentScreen.boundsLocked}
        <div class="bottom">
            <Button on:click={identifyScreens} title={$dictionary.settings?.identify_screens} style="padding: 12px;" dark center>
                <Icon id="search" size={1.4} white />
                <!-- <p><T id="settings.identify_screens" /></p> -->
            </Button>
        </div>
    {/if}
{/if}

<style>
    .tip {
        margin-bottom: 10px;

        opacity: 0.7;
        font-size: 0.8em;
        /* text-align: center; */
    }

    .content {
        width: 100%;
        display: flex;
        /* justify-content: center; */
        /* transform: translateX(-20%); */
    }
    .screens {
        --zoom: 0.08;
        zoom: var(--zoom);
        font-size: 20em;
        overflow: visible;
        /* position: relative;
        margin-top: auto; */
        position: absolute;
        inset-inline-start: 50%;
        top: calc(50% + 950px);
        /* transform: translateX(-1080px); */

        /* width: 30%;
    position: absolute;
    left: 50%;
    transform: translateX(-50%); */
    }

    .screens :global(.lock) {
        zoom: calc(1 / var(--zoom));
        position: absolute;
        top: 0;
        inset-inline-end: 0;

        /* pointer-events: initial; */
        padding: 5px !important;
        z-index: 2;
    }

    .bottom {
        position: absolute;
        bottom: 20px;
        inset-inline-start: 20px;
    }

    .screen {
        position: absolute;
        /* transform: translateX(-50%); */

        display: flex;
        align-items: center;
        justify-content: center;

        background-color: var(--primary);
        outline: 40px solid var(--primary-lighter);
        transition: background-color 0.1s;
    }

    .screen:hover:not(.disabled):not(.noClick) {
        background-color: var(--primary-lighter);
        cursor: pointer;
    }

    .screen.noClick {
        opacity: 0.5;
        pointer-events: none;
    }

    /* .screen.active, */
    .screen.noClick {
        outline: 40px solid var(--secondary);
        background-color: var(--primary-darker);
        z-index: 1;
    }

    .screen.disabled {
        opacity: 0.5;
    }
    .screen:focus:not(.disabled) {
        outline: 42px solid var(--secondary);
        outline-offset: 0;
    }

    .preview {
        display: flex;
        justify-content: center;
    }

    .border {
        background-color: black;

        outline: 2px solid var(--primary-lighter);
        outline-offset: 0;
        position: relative;

        display: flex;
        align-items: center;
        text-align: -webkit-center;
    }

    .cropped {
        position: absolute;
        width: 100%;
    }

    .previewWindow {
        background-color: var(--primary-darker);
        /* background-color: #ececec; */

        outline: 2px solid var(--secondary);
        outline-offset: 0;
    }
</style>
