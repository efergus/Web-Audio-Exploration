<script lang="ts">
	import { PlayerWithFilter } from '$lib/audio/player_with_filter';
	import { DEFAULT_AUDIO_SAMPLERATE, SampleData } from '$lib/audio/sample';
	import type { IirDigital } from '$lib/dsp/iir';
	import {
		chirpSample,
		phaseNoiseSample,
		pinkNoiseSample,
		whiteNoiseSample
	} from '$lib/dsp/samples';
	import PlayPixel from '$lib/icons/PlayPixel.svelte';
	import { Span1D, span1d, span2d, span2dFromSpans, type Span2D } from '$lib/math/span';
	import Button from '../input/Button.svelte';
	import AudioFileInput from './AudioFileInput.svelte';
	import Recorder from './Recorder.svelte';
	import Spectrogram from './Spectrogram.svelte';
	import Waveform from './Waveform.svelte';

	let {
		data = $bindable(new SampleData()),
		filteredData,
		filter,
		cursor = $bindable(null),
		span = $bindable(span2d(0, 1, -1, 1)),
		frequencySpan = $bindable(span1d(0, data.samplerate / 2)),
		playing = false,
		onData
	}: {
		data?: SampleData;
		filteredData?: SampleData;
		filter?: IirDigital;
		span?: Span2D;
		cursor?: number | null;
		playing?: boolean;
		frequencySpan?: Span1D;
		onData?: (sample: SampleData) => void;
	} = $props();
</script>

<div class="stack">
	<div class="audio">
		<Waveform {data} {filteredData} bind:span bind:cursor height={250} {playing} />
		<Spectrogram
			height={250}
			data={filteredData ?? data}
			logScale
			bind:span={() => span2dFromSpans(span.x, frequencySpan),
			(newSpan) => {
				frequencySpan = newSpan.y;
				span = span2dFromSpans(newSpan.x, span.y);
			}}
			bind:cursor
			{playing}
		/>
	</div>
	<div class="buttons">
		<AudioFileInput
			onData={(sample) => {
				data = sample;
				onData?.(sample);
				const sampleSpan = sample.span();
				const vertical = Math.max(Math.abs(sampleSpan.y.min), Math.abs(sampleSpan.y.max));
				span = span2dFromSpans(sampleSpan.x, span1d(-vertical, vertical));
			}}
		/>

		<Recorder
			onData={(sample) => {
				data = sample;
				onData?.(sample);
			}}
		/>

		<div class="spacer"></div>

		<Button
			onclick={() => {
				data = chirpSample(20, 4000);
				onData?.(data);
			}}
			>Chirp
		</Button>
		<Button
			onclick={() => {
				data = whiteNoiseSample(4 * DEFAULT_AUDIO_SAMPLERATE);
				onData?.(data);
			}}
			>Noise
		</Button>
		<Button
			onclick={() => {
				data = pinkNoiseSample(4 * DEFAULT_AUDIO_SAMPLERATE);
				onData?.(data);
			}}
			>Pink noise
		</Button>
		<Button
			onclick={() => {
				data = phaseNoiseSample(4 * DEFAULT_AUDIO_SAMPLERATE);
				onData?.(data);
			}}
			>Random phase
		</Button>
	</div>
</div>

<style lang="less">
	.stack {
		display: flex;
		flex-direction: column;
		max-width: min-content;
		gap: 6px;

		> :not(:last-child) {
			border-bottom: none;
		}
	}

	.audio {
		display: flex;
		flex-direction: column;
		justify-content: space-between;
		// border: 1px solid black;

		gap: 6px;
	}

	.buttons {
		display: flex;
		justify-content: stretch;
		border: 1px solid black;

		> :global(*) {
			border: none;
		}
		> :global(:not(:last-child)) {
			border-right: 1px solid silver;
		}
		> :global(div.spacer) {
			flex-grow: 1;
			margin: 0px;
		}
	}
</style>
