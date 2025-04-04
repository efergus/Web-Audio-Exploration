<script lang="ts">
	import { DEFAULT_AUDIO_SAMPLERATE, SampleData, type Sample } from '$lib/audio/sample';
	import Tape from '$lib/components/audio/Tape.svelte';
	import IirFilterEditor from '$lib/components/filters/IirFilterEditor.svelte';
	import { squareSample } from '$lib/dsp/samples';
	import { clamp } from '$lib/math/clamp';
	import { isClose } from '$lib/math/float';
	import { point } from '$lib/math/point';
	import { Span2D, span2d } from '$lib/math/span';
	import { onMount } from 'svelte';

	const initialDuration = 2;
	const initialSample = squareSample(
		DEFAULT_AUDIO_SAMPLERATE / 100,
		DEFAULT_AUDIO_SAMPLERATE,
		DEFAULT_AUDIO_SAMPLERATE * initialDuration,
		0.8
	);
	let data: SampleData = $state(initialSample);
	let lastData: SampleData = $state(initialSample);
	let filteredData: SampleData = $state(initialSample);

	const window = 256 / DEFAULT_AUDIO_SAMPLERATE;
	let span = $state(span2d(0, window, -1, 1));
	const minDuration = 1 / DEFAULT_AUDIO_SAMPLERATE;
	let effectiveLimits = $state(span2d(0, initialDuration, -100, 100));
	onMount(() => {
		let lastDuration = data.duration();
		const updateSpan = () => {
			const duration = data.duration();
			if (data !== lastData) {
				lastData = data;
				span = span2d(0, Math.min(span.x.size(), data.duration()), -1, 1);
				effectiveLimits = span2d(0, data.duration(), -100, 100);
			} else if (duration > lastDuration) {
				let window = Math.min(span.x.size(), duration);
				const start = Math.max(0, duration - window);
				span = span2d(start, start + window, -1, 1);
				effectiveLimits = span2d(0, Math.max(duration, window), -100, 100);
				lastDuration = duration;
			}
			requestAnimationFrame(updateSpan);
		};
		requestAnimationFrame(updateSpan);
	});

	let effectiveMinDuration = $derived(minDuration ?? 1 / data.samplerate);

	const getSpan = () => span;
	const setSpan = (newSpan: Span2D) => {
		const size = span.size();
		if (newSpan.x.size() < effectiveMinDuration) {
			const center = span.x.center();
			span = span2d(
				center - effectiveMinDuration / 2,
				center + effectiveMinDuration / 2,
				span.y.start,
				span.y.end
			);
			return;
		}

		if (
			size.x > 1e-6 &&
			isClose(size.x, newSpan.x.size(), 1e-9) &&
			isClose(size.y, newSpan.y.size(), 1e-9)
		) {
			const start = point(
				clamp(newSpan.x.start, effectiveLimits.x.start, effectiveLimits.x.end - size.x),
				clamp(newSpan.y.start, effectiveLimits.y.start, effectiveLimits.y.end - size.y)
			);
			span = span2d(start.x, start.x + size.x, start.y, start.y + size.y);
		} else {
			span = newSpan.intersect(effectiveLimits);
		}
	};

	$inspect(effectiveLimits);
</script>

<div>
	<Tape bind:span={getSpan, setSpan} {data} {filteredData} onData={(sample) => (data = sample)} />
	<IirFilterEditor
		{data}
		bind:span={getSpan, setSpan}
		onFilteredData={(sample) => (filteredData = sample)}
	/>
</div>

<style>
	div {
		display: flex;
		flex-direction: column;
		gap: 6px;
		padding: 6px;
	}
</style>
