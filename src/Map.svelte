<script>
import { paths } from "./norge-kommuner"

export let data
export let config
export let parties
export let output

console.log("Map:", config)

const getSpan = function(d, variable) {
    let values = d.map(v => v[variable])
    let min = Math.min(...values)
    let max = Math.max(...values)
    let span = max - min
    return {min, max, span}
}

$: getStyle = function(id) {
    let kommune = data.find(k => k.gammelid == id)
    if (kommune) {
        if (config.view == "best" && !config.party) {
            let best = Object.entries(kommune.partier).sort(([,a],[,b]) => b - a)
            let color = parties.find(p => p.id == best[0][0]).color
            return `fill: ${color};`
        }
        if (config.view == "mostProgress" && !config.party) {
            let best = Object.entries(kommune.endring).sort(([,a],[,b]) => b - a)
            let color = parties.find(p => p.id == best[0][0]).color
            return `fill: ${color};`
        }
        if (config.view == "mostProgress" && config.party) {
            let color = kommune.endring[config.party] > 0 ? parties.find(p => p.id == config.party).color : "#ccc"
            return `fill: ${color};`
        }
        if (config.view == "mostRegress" && !config.party) {
            let worst = Object.entries(kommune.endring).sort(([,a],[,b]) => a - b)
            let color = parties.find(p => p.id == worst[0][0]).color
            return `fill: ${color};`
        }
        if (config.view == "mostRegress" && config.party) {
            let color = kommune.endring[config.party] < 0 ? parties.find(p => p.id == config.party).color : "#ccc"
            return `fill: ${color};`
        }
    }
    return ""
}

</script>

{#if data.length > 0}
<svg xmlns="http://www.w3.org/2000/svg" version="1.2" viewBox="0 0 800 998" stroke-linecap="round" stroke-linejoin="round">
    <g>
    {#each paths as path}
    <path d={path.d} data-nr={path.nr} data-navn={path.navn} style="{getStyle(path.nr)}" />
    {/each}
    </g>
</svg>
{/if}

<style>
svg {
    height: 500px;
    margin-block: 20px;
}
path {
    stroke: #979797;
    stroke-width: .5px;
    fill: white;
}
</style>