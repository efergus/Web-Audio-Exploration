<script lang="ts">
	let {
		size = 24,
		width,
		height,
		fill = 'none',
		stroke = 'currentColor'
	}: {
		size?: number;
		width?: number;
		height?: number;
		fill?: string;
		stroke?: string;
	} = $props();

	width = width ?? height ?? size;
	height = height ?? width ?? size;
</script>

<svg
	{width}
	{height}
	viewBox="0 0 24 24"
	version="1.1"
	xmlns="http://www.w3.org/2000/svg"
	{fill}
	{stroke}
	shape-rendering="crispEdges"
>
	<rect width="20" height="20" x="2" y="2" />
</svg>
