<script lang="ts">
	import { SampleData, type Sample } from '$lib/audio/sample';
	import { filterRoots, IirDigital, single_pole_bandpass, type Root } from '$lib/dsp/iir';
	import { span2d, type Span1D, type Span2D } from '$lib/math/span';
	import PoleZeroEditor from '../../../routes/PoleZeroEditor.svelte';
	import FilterDetails from './FilterDetails.svelte';
	import { onMount } from 'svelte';
	import { PlayerWithFilter } from '$lib/audio/player_with_filter';
	import { IirRoots } from '$lib/state/roots.svelte';

	let {
		data,
		span = $bindable(),
		frequencySpan = $bindable(),
		onFilteredData
	}: {
		data: SampleData;
		span: Span2D;
		frequencySpan: Span1D;
		onFilteredData?: (sample: SampleData) => void;
	} = $props();

	const whatever = 0.1;
	const whatever2 = 0.1;
	const initialFilter = single_pole_bandpass(whatever, whatever2);
	const sample_digital_filter = $derived(initialFilter.to_digital_bilinear());
	let roots: IirRoots = $state(new IirRoots(filterRoots(initialFilter)));
	let previousInput: Sample | null = $state(null);
	let previousFilter: IirDigital | null = $state(null);
	let filteredData = $state(new SampleData());
	let filterChanged = $state(0);
	let player: PlayerWithFilter = $state(
		new PlayerWithFilter(undefined, {
			callback: ({ remaining }) => {
				if (!remaining) {
					const now = Date.now();
					if (now - filterChanged < 3000) {
						setTimeout(() => player.play(data), 250);
					}
				}
			}
		})
	);

	const digital_filter = $derived.by(() => {
		const baseFilter = IirDigital.from_roots(roots.zPlane, 1);
		const peakResponseFreq = baseFilter.max_frequency_response();
		const peakResponse = baseFilter.frequency_response_norm(peakResponseFreq);
		baseFilter.gain = 1 / peakResponse;
		return baseFilter;
	});
	const gain = $derived(digital_filter.gain);

	$effect(() => {
		roots.setSPlane(filterRoots(sample_digital_filter));
	});
	$effect(() => {
		player.setFilter(digital_filter);
		filterChanged = Date.now();
	});

	const updateFilteredData = (sample: SampleData, filter: IirDigital) => {
		let startIndex = 0;
		if (sample === previousInput && filter === previousFilter) {
			startIndex = filteredData.length;
		} else {
			previousInput = sample;
			previousFilter = filter;
			filteredData = new SampleData();
		}
		filteredData.push(
			filter.apply(sample.slice(startIndex), sample.slice(0, startIndex), filteredData)
		);
		onFilteredData?.(filteredData);
	};

	onMount(() => {
		let elapsed = 0;
		const doFilterUpdate = (dt: number) => {
			elapsed += dt;
			const quick = data === previousInput && digital_filter === previousFilter;
			if (quick || elapsed > 1000 / 10) {
				updateFilteredData(data, digital_filter);
				elapsed = 0;
			}
			requestAnimationFrame(doFilterUpdate);
		};
		requestAnimationFrame(doFilterUpdate);
	});

	$inspect(roots);
</script>

<div class="stack">
	<div style:height="250px">
		<p>Gain: {gain.toPrecision(3)}</p>
	</div>
	<div>
		<PoleZeroEditor
			bind:roots={() => roots.sPlane, (value) => roots.setSPlane(value)}
			span={span2d(-1, 0.1, frequencySpan.start, frequencySpan.end)}
		/>
		<PoleZeroEditor
			bind:roots={() => roots.zPlane, (value) => roots.setZPlane(value)}
			zPlane={true}
			span={span2d(-1.2, 1.2, -1.2, 1.2)}
		/>
		<FilterDetails filter={digital_filter} />
	</div>
</div>

<style lang="less">
	div.stack {
		flex-direction: row;
		gap: 6px;
	}
</style>
