<script>
import { afterUpdate, onMount } from "svelte"
import valg2019 from "./valg-2019.json"
import conversion from "./conversion.json"

const parties = [
    { id: "Ap", color: "#e4002b", name: "Arbeiderpartiet"},
    { id: "H", color: "#00458a", name: "Høyre"},
    { id: "R", color: "#630000", name: "Rødt"},
    { id: "SV", color: "#d94abf", name: "Sosialistisk Venstreparti"},
    { id: "Sp", color: "#69be28", name: "Senterpartiet"},
    { id: "MDG", color: "#008241", name: "Miljøpartiet De Grønne"},
    { id: "KrF", color: "#fcb211", name: "Kristelig Folkeparti"},
    { id: "V", color: "#006666", name: "Venstre"},
    { id: "Frp", color: "#22245d", name: "Fremskrittspartiet"},
    { id: "A", color: "#cccccc", name: "Andre"},
]
let data = {}
let config = {
    party: "",
    view: "best",
    length: 10
}
let partyStack = []
let output = []
let working = false
let mounted = false
let max

function mount(d) {
    data = d
    console.log("Mount:", d.data)
    for (const kommune of d.data) {
        // Valgdata for kommune
        let kommune2019
        if (valg2019.find(k => k.Id == kommune.kommuneid)) {
            kommune2019 = valg2019.find(k => k.Id == kommune.kommuneid)
        }
        else if (conversion.find(c => c[1] == kommune.kommuneid)) {
            let oldId = conversion.find(c => c[1] == kommune.kommuneid)[0]
            kommune2019 = valg2019.find(k => k.Id == oldId)
        }
        else {
            console.log("Mount: Kunne ikke finne " + kommune.kommuneid)
        }
        // Legg alle forekomster av partier til partystack
        for (const parti in kommune.partier) {
            partyStack.push({
                id: kommune.id,
                kommune: kommune2019 ? kommune2019.Navn : false,
                parti: parti,
                siste: Number(kommune.partier[parti]),
                valg: kommune2019 ? kommune2019[parti] : false,
                endring: kommune2019 ? kommune.partier[parti] - kommune2019[parti] : false
            })
        }
    }
    working = false
    mounted = true
}

function update(config) {
    console.log("Config:", config)
    if (!config.view && !config.party) return
    // Base
    output = partyStack
    if (config.party) output = output.filter(a => a.parti == config.party)
    // Most progress
    if (config.view == "mostProgress") {
        output = output
        .filter(a => a.valg != false)
        .filter(a => a.endring > 0)
        .sort((a,b) => b.endring - a.endring)
    }
    // Most regress
    else if (config.view == "mostRegress") {
        output = output
        .filter(a => a.valg != false)
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

// Helpers
function getColor(id) {
    return parties.find(p => p.id == id).color
}

</script>

<div class=nav>
    <select class:active={config.view} name="view" bind:value={config.view}>
        <option value="">Velg målemetode</option>
        <option value="best">Størst oppslutning for ...</option>
        <option value="mostProgress">Mest framgang for ...</option>
        <option value="mostRegress">Mest tilbakegang for ...</option>
        <option value="majority">Flertall for ...</option>
    </select>
    <select class:active={config.party} disabled={config.view == ""} name="party" bind:value={config.party}>
        <option value="">Alle partier</option>
        {#each parties as party}
        <option value="{party.id}">{party.name}</option>
        {/each}
    </select>
</div>

{#if output.length > 0}
<div class="list">
    {#each output as item, i}
    {#if i < config.length}
    <div>
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
    </div>
    {/if}
    {/each}
</div>
<div class=credits>Basert på lokale målinger for {data.data.length} av 357 kommuner. Hentet {new Date(data.time).toLocaleString("nb-NO")} fra <a href="https://pollofpolls.no">pollofpolls.no</a>.</div>
{:else if working}
<div class=notice>
Arbeider...
</div>
{:else if !config.view && !config.party}
<div class=notice>
    Velg ovenfor ☝️
</div>
{:else}
<div class=notice>
    Fant ingen partier.
</div>

{/if}

<style>
select {
    padding: 5px 10px;
    border-radius: 99px;
    font-size: 18px;
    font-weight: 500;
}
.notice {
    margin-block: 10px;
}
.list {
    display: flex;
    flex-direction: column;
    gap: 7.5px;
    margin-block: 20px;
}
.municipality {
    font-weight: bold;
}
.content {
    display: flex;   
    gap: 10px;
}
.name {
    width: 30px;
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
}
.bars > div:first-child {
    top: 0;
}
.bars > div:last-child {
    bottom: 0;
    opacity: .45
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
.credits {
    font-size: .9em;
}
</style>