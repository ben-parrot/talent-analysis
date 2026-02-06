<script>
	import '@evidence-dev/tailwind/fonts.css';
	import '../app.css';
	import { EvidenceDefaultLayout } from '@evidence-dev/core-components';
	export let data;
</script>

<EvidenceDefaultLayout 
    {data} 
    maxWidth={1600}
    darkLogo="/Parrot_Analytics_Logo_Dark_Background.png"
    lightLogo="/Parrot_Analytics_Logo_Light_Background.png"
    builtWithEvidence={false}
    downloadable={true}
    neverShowQueries={true}
    homePageName="NNAF Talent Analysis"
>
    <slot slot="content" />
</EvidenceDefaultLayout>

<style>
    :global(body) {
        font-size: 24px;
    }
</style>