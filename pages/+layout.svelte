<script>
	import '@evidence-dev/tailwind/fonts.css';
	import '../app.css';
	import { EvidenceDefaultLayout } from '@evidence-dev/core-components';
	export let data;
</script>

<EvidenceDefaultLayout {data} maxWidth={1600}>
    <slot slot="content" />
</EvidenceDefaultLayout>

<style>
    :global(body) {
        font-size: 24px;
    }
</style>