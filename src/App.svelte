<script>
// @ts-nocheck

import { afterUpdate, onMount } from "svelte"
import valg2019 from "./valg-2019.json"
import conversion from "./conversion.json"
import Map from "./Map.svelte"

const parties = [
    { id: "Ap", color: "#d70926", name: "Arbeiderpartiet"},
    { id: "H", color: "#0065f1", name: "Høyre"},
    { id: "R", color: "#630000", name: "Rødt"},
    { id: "SV", color: "#d94abf", name: "Sosialistisk Venstreparti"},
    { id: "Sp", color: "#00843d", name: "Senterpartiet"},
    { id: "MDG", color: "#4d7e00", name: "Miljøpartiet De Grønne"},
    { id: "KrF", color: "#fcb211", name: "Kristelig Folkeparti"},
    { id: "V", color: "#006666", name: "Venstre"},
    { id: "Frp", color: "#003955", name: "Fremskrittspartiet"},
    { id: "A", color: "#aaaaaa", name: "Andre partier"},
]
let data = {}
let count
$: if(data.data) count = data.data.length
let config = {
    party: "",
    mode: "list",
    view: "",
    length: 100
}
var dateOptions = {
    year: "numeric",
    month: "short",
    day: "numeric"
};
let partyStack = []
let output = []
let working = false
let mounted = false
let max
let expandInfo

function mount(d) {
    data = d
    console.log("Mount:", d.data)
    for (const kommune of d.data) {
        // Valgdata for kommune
        let kommune2019
        if (valg2019.find(k => k.Id == kommune.kommuneid)) { // Hvis gammel id, hent data
            kommune2019 = valg2019.find(k => k.Id == kommune.kommuneid) 
        }
        else if (conversion.find(c => c[1] == kommune.kommuneid)) { // Hvis ny id, hent gammel id og hent data
            let oldId = conversion.find(c => c[1] == kommune.kommuneid)[0]
            kommune2019 = valg2019.find(k => k.Id == oldId)
        }
        else {
            console.log("Mount: Kunne ikke finne " + kommune.kommuneid)
        }
        kommune.endring = {}
        // Legg alle forekomster av partier til partystack
        for (const parti in kommune.partier) {
            partyStack.push({
                id: kommune.id,
                kommune: kommune2019 ? kommune2019.Navn : false,
                kommunenr: kommune2019 ? kommune2019.Id : kommune.kommuneid,
                parti: parti,
                siste: Number(kommune.partier[parti]),
                valg: kommune2019 ? kommune2019[parti] : false,
                endring: kommune2019 ? kommune.partier[parti] - kommune2019[parti] : false
            })
            kommune.endring[parti] = kommune2019 ? (kommune.partier[parti] - kommune2019[parti]).toFixed(2) : false
        }
        // Legg til kommuneinfo i data
        kommune.navn = kommune2019 ? kommune2019.Navn : ""
        kommune.gammelid = kommune2019 ? kommune2019.Id : kommune.kommuneid
    }
    working = false
    mounted = true
}

function update(config) {
    expandInfo = undefined
    console.log("Config:", config)
    if (!config.view && !config.party) return
    // Base
    output = partyStack
    if (config.party) output = output.filter(a => a.parti == config.party)
    // Most progress
    if (config.view == "mostProgress") {
        output = output
        // .filter(a => a.valg != false)
        .filter(a => a.endring > 0)
        .sort((a,b) => b.endring - a.endring)
    }
    // Most regress
    else if (config.view == "mostRegress") {
        output = output
        // .filter(a => a.valg != false)
        .filter(a => a.endring < 0)
        .sort((a,b) => a.endring - b.endring)
    }
    else if (config.view == "majority") {
        output = output
        .filter(a => a.siste >= 50)
        .sort((a,b) => b.siste - a.siste)
    }
    else if (config.view == "best") {
        output = output
        .sort((a,b) => b.siste - a.siste)
    }
    // Add meta
    output.map(a => {
        a.meta = data.data.find(b => b.id == a.id)
        return a
    })
    max = Math.max(...output.map(e => e.siste), ...output.map(e => e.valg))

    output = output
    console.log("Output:",output)
}

onMount(() => {
    working = true
    const url = "https://valg2023-polls.vercel.app/api/get"
    fetch(url)
    .then(r => r.json())
    .then(d => {
        mount(d)
    })
})

$: if(mounted) update(config)

function expand(i) {
    if (expandInfo && i == expandInfo.i) expandInfo = undefined
    else {
        expandInfo = output[i].meta
        expandInfo.i = i
    }
}

// Helpers
function getColor(id) {
    return parties.find(p => p.id == id).color
}
function reformatString(string) {
    return string
    .replace("Ã¸", "ø")
    .replace("Ã¥", "å")
    .replace("Ã¦", "æ")
}


</script>
<div class=container>
    <div class=nav>
        <select class:active={config.view} name="view" bind:value={config.view}>
            <option value="">Velg liste</option>
            <option value="best">🏆 Størst oppslutning blant</option>
            <option value="mostProgress">📈 Størst framgang blant</option>
            <option value="mostRegress">📉 Størst tilbakegang blant</option>
            <option value="majority">⚖️ Flertall for</option>
        </select>
        <select class:active={config.party} disabled={config.view == ""} name="party" bind:value={config.party}>
            <option value="">Alle partier</option>
            {#each parties as party}
            <option value="{party.id}">{party.name}</option>
            {/each}
        </select>
        {#if config.view}
        <div class=toggle-map on:click={() => {config.mode = config.mode == "list" ? "map" : "list"}} on:keypress={() => {config.mode = config.mode == "list" ? "map" : "list"}} role=button tabindex=0>Vis {config.mode == "list" ? "kart" : "liste"}</div>
        {/if}
        {#if (config.view == "mostProgress" || config.view == "mostRegress") && config.party != ""}
        <div class=info>
            <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 50 50"><circle cx="25" cy="25" r="25" fill="#fcda51"></circle><circle cx="25" cy="14" r="3"></circle><rect x="22.5" y="20" width="5" height="18"></rect></svg>
            {parties.find(p => p.id == config.party).name} har {config.view == "mostProgress" ? "framgang" : "tilbakegang"} i {output.length} kommuner.</div>
        {/if}
    </div>
    {#if output.length > 0 && config.view}
    {#if config.mode == "list"}
    <div class="list">
        {#each output as item, i}
        {#if i < config.length}
        <!-- svelte-ignore a11y-no-static-element-interactions -->
        <div on:click={() => {expand(i)}} on:keypress={() => {expand(i)}}>
            <div class=municipality>{item.kommune}</div>
            <div class=content>
                <div class=name>{item.parti}</div>
                <div class=bars style="--color: {getColor(item.parti)};">
                    <div style="width:{item.siste / max * 98}%"></div>
                    <div style="width:{item.valg / max * 98}%"></div>
                </div>
                <div class=latest>{item.siste.toLocaleString("nb-NO", {maximumFractionDigits: 1})}%</div>
                <div class=valg-2019>{item.valg.toLocaleString("nb-NO", {maximumFractionDigits: 1})}%</div>
                <div class=diff class:negative={item.endring < 0}>{item.endring > 0 ? "+" : ""}{item.endring.toLocaleString("nb-NO", {maximumFractionDigits: 1})}</div>
            </div>
            {#if expandInfo && expandInfo.i == i}
            <div class=expandInfo>
                <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 50 50"><circle cx="25" cy="25" r="25" fill="#fcda51"></circle><circle cx="25" cy="14" r="3"></circle><rect x="22.5" y="20" width="5" height="18"></rect></svg>
                <div>Publisert {new Date(expandInfo.dato).toLocaleString("nb-NO", dateOptions)} i <a href="{expandInfo.url}" target="_blank">{reformatString(expandInfo.kilde)}</a>. Utført av {expandInfo.institutt} med {expandInfo.spurte} deltakere.</div>
            </div>
            {/if}
        </div>
        {/if}
        {/each}
    </div>
    {:else if config.mode == "map"}
    <Map data={data.data} config={config} parties={parties} output={output} />
    {/if}
    <div class=credits>Basert på lokale målinger for {data.data.length} av 357 kommuner. Hentet {new Date(data.time).toLocaleString("nb-NO", dateOptions)} fra <a href="https://pollofpolls.no">pollofpolls.no</a>. </div>
    {:else if working}
    <div class=notice>
    Arbeider...
    </div>
    {:else if !config.view}
    <div class=notice>
        <h3>Velg ovenfor ☝️</h3>
        <p>Denne oversikten bygger på meningsmålinger gjort på <strong>kommunenivå</strong> siden 1. juli 2023.</p>
        <p>Ikke alle aviser gjør egne, lokale målinger, og nasjonale eller fylkesvise målinger er ikke presise nok til å si noe om hvordan folk vil stemme i hver kommune.</p>
        <p>Dersom det finnes flere målinger i en kommune, er det den siste målingen som vises.</p>
        {#if typeof count == "number"}
        Akkurat nå er det <strong>{count}</strong> av 357 ({(count / 357 * 100).toFixed(1)}%) kommuner i oversikten.
        {/if}
    </div>
    {:else}
    <div class=notice>
        Ingen partier/kommuner.
    </div>
    {/if}
</div>

<style>
.container {
    height: 100%;
    display: flex;
    flex-direction: column;
}
a {
    color: black;
}
svg {
    width: 20px;
    min-width: 20px;
}
.nav {
    display: flex;
    flex-wrap: wrap;
    align-items: center;
    justify-content: center;
    gap: 10px;
}
select {
    padding: 10px 20px;
    border-radius: 99px;
    font-size: 16px;
    font-weight: 500;
    outline: none;
    background-color: transparent;
    cursor: pointer;
}
.toggle-map {
    padding: 10px 20px;
    border-radius: 99px;
    font-weight: 500;
    border: 1px solid #666;
    cursor: pointer;
}
.info {
    padding: 10px 20px;
    border-radius: 99px;
    background: #eee;
    display: flex;
    gap: 10px;
    align-items: center;
}
.notice {
    margin: 25px auto;
    width: 80%;
    text-align: center;
}
.list {
    display: flex;
    flex-direction: column;
    gap: 8px;
    margin-block: 20px 10px;
    overflow-y: scroll;
}
.list > div {
    cursor: help;
}
.municipality {
    font-weight: bold;
}
.content {
    display: flex;   
    gap: 10px;
    align-items: center;
    margin-top: 4px;
}
.name {
    width: 35px;
}
.bars {
    position: relative;
    height: 24px;
    flex: 1;
}
.bars > div {
    position: absolute;
    left: 0;
    height: 60%;
    background: var(--color);
    border-bottom-right-radius: 99px;
    border-top-right-radius: 99px;
}
.bars > div:first-child {
    top: 0;
}
.bars > div:last-child {
    bottom: 0;
    opacity: .4
}
.latest,
.valg-2019,
.diff {
    width: 45px;
    text-align: right;
}
.latest {
    font-weight: bold;
}
.valg-2019 {
    font-weight: 200;
}
.diff {
    font-weight:bold;
    color: green;
}
.diff.negative {
    color: red;
}
.list > div:first-child .latest,
.list > div:first-child .valg-2019,
.list > div:first-child .diff {
    position: relative;
}
.list > div:first-child .latest:before,
.list > div:first-child .valg-2019:before,
.list > div:first-child .diff:before {
    position: absolute;
    top: -30px;
    right: 0;
    content: "Siste";
    font-size: 13px;
    font-weight: normal;
}
.list > div:first-child .valg-2019:before {
    content: "2019";
}
.list > div:first-child .diff:before {
    content: "+/-";
    color: black;
}
.expandInfo {
    font-size: .9em;
    margin-block: 10px;
    padding: 10px 20px;
    background: #eee;
    border-radius: 99px;
    display: inline-flex;
    justify-content: flex-start;
    gap: 10px;
}
.credits {
    font-size: .9em;
    color: #666;
}
</style>