<script lang="ts">
	import functionPlot from 'function-plot';

	type method = 'rectangleLeft' | 'rectangleRight' | 'rectangleMidpoint' | 'trapezoid' | 'simpson';

	interface SingleEquation {
		id: number;
		label: string;
		fn: string;
	}

	interface SolveResponse {
		value: number;
		partitions: number;
	}

	interface Payload {
		functionId: number;
		method: method;
		a: number;
		b: number;
		tolerance: number;
	}

	const singleEqs: SingleEquation[] = [
		{
			"id": 0,
			"label": "f(x) = -x³ - x² + x + 3",
			"fn": "-x^3 - x^2 + x + 3.0"
		},
		{
			"id": 1,
			"label": "f(x) = e^(-x²)",
			"fn": "exp(-x*x)"
		},
		{
			"id": 2,
			"label": "f(x) = 1/√|x|",
			"fn": "1 / sqrt(abs(x))"
		},
		{
			"id": 3,
			"label": "f(x) = 1/x",
			"fn": "1 / x"
		},
		{
			"id": 4,
			"label": "f(x) = 1/√|2x - x²|",
			"fn": "1 / sqrt(abs(2*x - x*x))"
		}
	];

	let selectedSingleEq = $state<number>(0);

	let a = $state<number>(-5);
	let b = $state<number>(5);
	let method = $state<method>('rectangleLeft');

	let tolerance = $state<number>(0.001);

	let result = $state<SolveResponse | null>(null);
	let isLoading = $state<boolean>(false);
	let errorMsg = $state<string | null>(null);

	let plotContainer: HTMLDivElement | undefined = $state();

	$effect(() => {
		if (plotContainer) {
			const sEq = selectedSingleEq;
			drawPlot();
		}
	});

	function drawPlot(): void {
		if (!plotContainer) return;

		try {
			const data: any[] = [
				{ 
					fn: singleEqs[selectedSingleEq].fn, 
					color: '#ff5722', 
					nSamples: 5000, 
					range: [a, b],
					closed: true
				}, 
				{
					fn: singleEqs[selectedSingleEq].fn, 
					color: 'red', 
					nSamples: 5000, 
				}
			];

			functionPlot({
				target: plotContainer,
				width: 600,
				height: 400,
				grid: true,
				xAxis: { domain: [-10, 10] },
				yAxis: { domain: [-10, 10] },
				data
			});
		} catch (err) {
			console.error("Ошибка построения графика:", err);
		}
	}

	async function handleSubmit(): Promise<void> {
		isLoading = true;
		errorMsg = null;
		result = null;

		let payload: Payload;

		payload = {
			functionId: selectedSingleEq,
			method: method,
			a,
			b,
			tolerance,
		};

		try {
        const res = await fetch(`/api/solve`, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify(payload)
        });

        const data = await res.json();

        if (!res.ok) {
            throw new Error(`Ошибка: ${data.error}` || `Ошибка сервера: ${res.status}`);
        }
        
        result = data as SolveResponse;

		} catch (err) {
			errorMsg = err instanceof Error ? err.message : String(err);
		} finally {
			isLoading = false;
		}
	}
</script>

<main class="container">
	<h1>Вычислительная математика 3 Зенченков P3215</h1>

	<div class="layout">
		<div class="controls">
			<div class="section">
				<h3>Выбор уравнения</h3>
				{#each singleEqs as eq}
					<label>
						<input type="radio" bind:group={selectedSingleEq} value={eq.id}> {eq.label}
					</label>
				{/each}
			</div>

			<div class="section">
				<h3>Метод решения</h3>
				<select bind:value={method}>
					<option value="rectangleLeft">Метод левых прямоугольников</option>
					<option value="rectangleRight">Метод правых прямоугольников</option>
					<option value="rectangleMidpoint">Метод средних</option>
					<option value="trapezoid">Метод трапеций</option>
					<option value="simpson">Метод Симпсона</option>
				</select>
			</div>

			<div class="section">
				<h3>Параметры</h3>
				<div class="input-group">
					<label>Интервал [a]: <input type="number" step="0.1" bind:value={a}></label>
					<label>Интервал [b]: <input type="number" step="0.1" bind:value={b}></label>
				</div>
				<label style="margin-top: 10px; display: block;">
					Погрешность (ε): <input type="number" min="0" step="0.001" bind:value={tolerance}>
				</label>
			</div>

			<button onclick={handleSubmit} disabled={isLoading}>
				{isLoading ? 'Вычисляем...' : 'Решить'}
			</button>

			{#if result}
				<div class="result success">
					<h4>Результат: {result.value}</h4>
					<p><b>Количество делений:</b> {result.partitions}</p>
				</div>
			{/if}

			{#if errorMsg}
				<div class="result error">
					<p>{errorMsg}</p>
				</div>
			{/if}
		</div>

		<div class="chart">
			<h3>График</h3>
			<div bind:this={plotContainer}></div>
		</div>
	</div>
</main>

<style>
	.container { max-width: 1000px; margin: 0 auto; padding: 20px; font-family: sans-serif; }
	.layout { display: flex; gap: 40px; }
	.controls { flex: 1; }
	.chart { flex: 1; display: flex; flex-direction: column; align-items: center; }
	.section { margin-bottom: 20px; padding: 15px; border: 1px solid #ddd; border-radius: 8px; }
	h3 { margin-top: 0; margin-bottom: 15px; font-size: 16px; color: #333; }
	label { display: block; margin-bottom: 8px; cursor: pointer; }
	.input-group { display: flex; gap: 10px; }
	.input-group label { display: flex; flex-direction: column; font-size: 14px; }
	input[type="number"], select { padding: 6px; margin-top: 4px; border: 1px solid #ccc; border-radius: 4px; }
	button { padding: 10px 20px; background: #007bff; color: white; border: none; border-radius: 4px; font-size: 16px; cursor: pointer; width: 100%; }
	button:hover { background: #0056b3; }
	button:disabled { background: #aaa; cursor: not-allowed; }
	.result { margin-top: 20px; padding: 15px; border-radius: 8px; }
	.success { background: #e8f5e9; border: 1px solid #c8e6c9; }
	.error { background: #ffebee; border: 1px solid #ffcdd2; color: #c62828; }
</style>