<script>
    import NotificationsPerm from "./NotificationsPerm.svelte";
    import TagEntry from "./TagEntry.svelte";
    import download from "../download.ts";
    import { rebuildTagIndex } from "../tagIndex.ts";
    import { onMount } from "svelte";
    import { toDurString, updatePint } from "../pint-update.ts";
    import { allPings, putPings } from "../pings.ts";
    import debounce from "../debounce.ts";
    import config from "../../config.json";

    if (window.loginState === "out") {
        location.href = "/";
    }

    const clearFail = () => alert("All data on this device has been deleted. However, there was an error deleting data from the server. As such, the data from the server may be re-synced once server communication is possible again. Please try deleting data again later.");
    async function deleteAllData(event) {
        const reallySure = confirm("Do you *really* want to delete everything? This will clear all data from your device and from the server.");
        if (!reallySure) return;
        localStorage.clear();
        try { await window.db.delete(); } catch (e) { console.log("Error clearing DB", e); }
        let res;
        try {
            res = await fetch(config["api-server"] + "/db", {
                method: "DELETE",
                credentials: "include",
            });
            if (!(res.status === 204 || res.status === 200)) clearFail();
        } catch (e) {
            clearFail();
        }
        location.href = "/";
    }

    const DAY_NAMES = ["SUN", "MON", "TUE", "WED", "THU", "FRI", "SAT"];
    function genLine(r) {
        let line = `${r.time} ${r.tags.join(" ")}`;
        let extraSpaces = Math.max(1, 54 - line.length);
        line += " ".repeat(extraSpaces);
        const d = new Date(r.time * 1000);
        // we always use the full form here,
        // although actual TagTime will sometimes use shorter forms
        // (to ensure all lines are the same length)
        line += `[${d.getFullYear()}.${(d.getMonth() + 1).toString().padStart(2, "0")}.${d.getDate().toString().padStart(2, "0")} ${d.toTimeString().split(" ")[0]} ${DAY_NAMES[d.getDay()]}]`;
        return line;
    }

    let tagtimeExportPending = false;
    async function tagtimeExport() {
        tagtimeExportPending = true;
        const rows = await allPings();
        if (rows === null) {
            alert("Failed to fetch pings from server. Ensure you are connected to the Internet.");
            return;
        }
        const data = rows
            .map(r => genLine(r))
            .join("\n");
        download(data, "text/plain", "tags.log");
        tagtimeExportPending = false;
    }

    function tryToPersist() {
        navigator.storage.persist();
    }

    function updatePingNoise() {
        localStorage["retag-audio"] = this.value;
        try {
            const ele = document.getElementById(`audio-${this.value}`);
            ele.play();
        } catch (e) { }
    }
    let pingNoise;
    onMount(() => {
        pingNoise = localStorage["retag-audio"] || "n";
    });

    let tagtimeImportPending = false;
    let ttImportButton;
    function endImport() {
        ttImportButton.value = "";
        tagtimeImportPending = false;
    }
    async function tagtimeImport() {
        if (ttImportButton.files.length === 0) return;
        tagtimeImportPending = true;
        const file = ttImportButton.files[0];
        // could use streams to improve performance, don't think it's needed though
        const text = await file.text();
        const allPings = text.split("\n").map(line => {
            line = line.trim();
            let parts = line.split(" ");
            if (parts.length < 2) return null; // at least "123 ["
            let time = parseInt(parts[0], 10);
            if (Number.isNaN(time)) return null;
            const humanTimeIndex = parts.findIndex(part => part.includes("["));
            const tags = parts
                .slice(1, humanTimeIndex)
                .map(tag => tag.trim().toLowerCase())
                .filter(tag => !tag.match(/^\W*$/));
            return {
                tags,
                time,
                category: null,
                interval: window.pintData.avg_interval,
                comment: null,
            };
        }).filter(ping => ping !== null);
        if (allPings.length === 0) {
            alert("No valid pings found.");
            endImport();
            return;
        }
        console.log(allPings);
        await putPings(allPings);
        await rebuildTagIndex();
        alert(`Imported ${allPings.length} pings.`);
        endImport();
        location.reload();
    }
    function doTtImport() {
        ttImportButton.click();
    }
    let theme = localStorage["retag-theme"];
    function updateTheme() {
        theme = this.value;
        localStorage["retag-theme"] = theme;
        location.reload();
    }

    let pintAvgInterval = toDurString(window.pintData.avg_interval);
    let pintSeed = window.pintData.seed.toString();
    let pintAlgChecked = localStorage["retag-pint-alg"] === "tagtime";
    function updatePintClick() {
        updatePint(pintAvgInterval, pintSeed, pintAlgChecked);
    }

    function useUnivSched() {
        updatePint("45:00", "1184097393", true);
    }

    let afkTags = (localStorage["retag-afk-tags"] || "afk").split(" ");
    if (afkTags.length === 1 && afkTags[0] === "") afkTags = [];
    const afkTagsUpdate = debounce(event => {
        afkTags = event.detail;
        localStorage["retag-afk-tags"] = afkTags.join(" ");
    }, 1750);

    function dbDownload() {
        location.href = `${config["api-server"]}/db`;
    }
</script>

<style>
    h2,
    h3 {
        margin: 0;
        margin-top: 0.8rem;
    }

    .dz,
    .delete-button {
        color: rgb(141, 0, 0);
    }

    :global(.dark) .dz,
    :global(.dark) .delete-button {
        color: rgb(247, 2, 2);
    }

    .ding-info,
    .ding-audio-info {
        color: rgb(56, 56, 56);
    }

    :global(.dark) .ding-info,
    :global(.dark) .ding-audio-info {
        color: rgb(197, 197, 197);
    }

    .tt-import {
        /* apparently some browsers don't like display: none inputs */
        position: absolute;
        left: -1000px;
        top: -1000px;
    }

    #pint-interval {
        width: 3em;
    }

    #pint-seed {
        width: 10em;
    }

    .adv-settings {
        display: inline;
    }

    .adv {
        padding-top: 1rem;
    }
</style>

<h2>Settings</h2>
<main id="maincontent">
    <div>
        Note: some settings cannot be changed while offline
    </div>
    <h3>Ping notifications</h3>
    <div>
        <NotificationsPerm />
    </div>
    <div>
        <label for="ping-noise">Audio to play on a ping: </label>
        <!-- I *this* using `change` is okay here  -->
        <!-- svelte-ignore a11y-no-onchange -->
        <select id="ping-noise" bind:value={pingNoise} on:change={updatePingNoise}>
            <option value="n">None</option>
            <option value="d3">Simple ding</option>
            <option value="d1">Bell Arpeggio</option>
            <option value="d2">Doorbell</option>
        </select>
        <div class="ding-audio-info">
            {#if pingNoise === "n"}
                No audio will be played.
            {:else if pingNoise === "d1"}
                It kinda sounds like some windchimes.
                Apparently an &ldquo;arpeggio&rdquo; is a type of broken chord.
                And a &ldquo;broken chord&rdquo; is a chord but broken up and maybe with some notes repeated.
                Audio is from <a href="https://cynicmusic.com/" rel="noopener">The Cynic Project</a>.
            {:else if pingNoise === "d2"}
                Sounds like a doorbell. Ding-dong! Audio is from
                <a href="https://freesound.org/people/MatthewWong/" rel="noopener" title="Don't get too excited about clicking this link. MatthewWong's only upload to freesound.org is this doorbell sound. Which you've already heard if you're hovering over this link.">MatthewWong</a>.
            {:else if pingNoise === "d3"}
                Sounds a bit futuristic (but not too futuristic).
                Audio is from
                <a href="https://freesound.org/people/robni7/" rel="noopener">robni7</a>.
            {:else if pingNoise === "d1337"}
                no
            {:else}
                Erm what? honestly I don't what know *what* will be played
            {/if}
        </div>
        <div class="ding-info">
            Select a ping noise to hear it.
            {#await window.supportsAutoplay then supported}
                {#if !supported && (pingNoise !== "n")}
                    <div class="autoplay-warning">
                        Note: Your browser might not support playing ping noises in some circumstances.
                    </div>
                {/if}
            {/await}
        </div>
    </div>

    <h3>Pinging</h3>
    <div>
        <div>
            <label for="pint-interval">
                Average ping interval (format like 45:12 for a ping every 45 minutes and 12 seconds, changing this will disable notifications):
            </label>
            <input type="text" id="pint-interval" bind:value={pintAvgInterval}>
        </div>
        <div>
            {#if pintAlgChecked && (!(pintAvgInterval.trim() === "45:00" || pintAvgInterval.trim() === "45:0") || (pintSeed !== "1184097393"))}
                Note: You are are using the original TagTime algorithm but not the universal schedule.
                Performance will be degraded and notifications will not be sent (due to the lack of lookup tables).
                Click the below button to use the universal schedule.
            {/if}
        </div>
        <div>
            Note: To prevent incorrect ping timing, reload all open TagTime Web tabs after changing this setting (while connected to the Internet).
        </div>
        <div>
            <button on:click={updatePintClick}>Update</button>
        </div>
    </div>
    <div>
        Default tags when <abbr title="Away From Keyboard">AFK</abbr>:
        <TagEntry bind:tags={afkTags} on:input={afkTagsUpdate} small />
    </div>

    <h3>Universal Schedule (beta)</h3>
    <div>
        <div>
            Click the button to use the
            <a href="https://forum.beeminder.com/t/official-reference-implementation-of-the-tagtime-universal-ping-schedule/4282">universal schedule</a>.
            This overides your interval, seed (in advanced settings), and algorithm (in advanced settings) to the universal schedule with just one click.
        </div>
        <div>
            Note: To prevent incorrect ping timing, reload all open TagTime Web tabs after changing this setting (while connected to the Internet).
        </div>
        <div>
            <button on:click={useUnivSched}>
                Use the universal schedule
            </button>
        </div>
    </div>

    <h3>Import/export</h3>
    <div>
        <label for="tt-import" class="tt-import-btn">
            <button on:click={doTtImport} disabled={tagtimeImportPending}>
                Import from TagTime
            </button>
        </label>
        (Only adds pings, does not remove or overwrite existing pings. Invalid lines are ignored. Does not import interval settings. Assumes TagTime used the currently active interval)
        <input id="tt-import" class="tt-import" aria-hidden="true" type="file" on:change={tagtimeImport} bind:this={ttImportButton} disabled={tagtimeImportPending}>
    </div>
    <div>
        <button on:click={tagtimeExport} disabled={tagtimeExportPending}>Export as TagTime log</button>
    </div>
    <div>
        <button on:click={dbDownload}>Download SQLite database</button>
    </div>

    <h2><label for="theme-dropdown">Theme</label></h2>
    <select id="theme-dropdown" on:input={updateTheme}>
        <option selected={theme === "default"} value="default">Browser default</option>
        <option selected={theme === "dark"} value="dark">Dark</option>
        <option selected={theme === "light"} value="light">Light</option>
    </select>

    <h2>Account</h2>
    <div>
        <a href={config["api-server"] + "/internal/changepw"}>Change password</a>
    </div>

    <h3 class="dz">Danger Zone</h3>
    <div>
        <button on:click={deleteAllData} class="delete-button">Delete all data</button>
    </div>
    <div>
        Note: This will not delete your account. To delete your account, <a href={"mailto:" + config["contact-email"]}>contact support</a>.
    </div>

    <details class="adv">
        <summary><h3 class="adv-settings">Advanced</h3></summary>
        <div>
            You probably don't want to touch any of the settings here.
        </div>
        <div>
            <button on:click={tryToPersist}>Request persistent storage</button>
        </div>
        <div>
            <button on:click={rebuildTagIndex}>Rebuild tag index</button>
        </div>
        <div>
            <label for="pint-seed">
                Ping seed (changing this will disable notifications, see above warning for interval):
            </label>
            <input type="number" id="pint-seed" bind:value={pintSeed}>
            <div>
                Note that if the box below is checked, performance will be worse if the seed isn't 1184097393. (due to lookup tables)
            </div>
            <button on:click={updatePintClick}>Update</button>
        </div>
        <div>
            <input type="checkbox" id="pint-tt-alg" bind:checked={pintAlgChecked}>
            <label for="pint-tt-alg">
                Use original TagTime algorithm (beta, see below)
            </label>
            <details>
                <summary>Read this before changing the above checkbox!</summary>
                You should check this box if you want compatability with the original TagTime.
                Checking this checkbox will enable using the algorithm used by the original TagTime.
                You can change the seed under advanced settings. Note that UR_PING is always 1184097393.
                To use the universal schdule, check the above checkbox, set the ping inverval to 45:00, and set the seed to 11193462 (or click the above button to do that for you).
            </details>
            <button on:click={updatePintClick}>Update</button>
        </div>
    </details>
</main>
