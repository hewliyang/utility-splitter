<script lang="ts">
	import { addDays, differenceInDays, isWithinInterval, parseISO } from 'date-fns';

	type Tenant = {
		name: string;
		absences: { start: string; end: string }[];
	};

	type BillSplit = {
		[key: string]: number;
	};

	type Summary = {
		total_w_gst: number;
		total_wo_gst: number;
	};

	let bill_start = $state('');
	let bill_end = $state('');
	let var_amt = $state(0);
	let fixed_amt = $state(0);
	let idle_cost_percent = $state(10);
	let gst_percent = $state(9);
	let tenants = $state<Tenant[]>([
		{ name: 'Jean', absences: [] },
		{ name: 'Hew', absences: [] },
		{ name: 'Kah Meng', absences: [] }
	]);

	function add_absence(idx: number) {
		tenants[idx].absences.push({ start: '', end: '' });
	}

	function update_absence(idx: number, absence_idx: number, field: 'start' | 'end', value: string) {
		tenants[idx].absences[absence_idx][field] = value;
	}

	let bill_split = $derived.by(() => {
		const start = parseISO(bill_start);
		const end = parseISO(bill_end);
		const total_days = differenceInDays(end, start) + 1;
		const daily_rate = var_amt / total_days;
		const idle_daily_rate = daily_rate * (idle_cost_percent / 100);
		const active_daily_rate = daily_rate - idle_daily_rate;

		const fixed_cost_per_person = fixed_amt / tenants.length;
		const split: BillSplit = Object.fromEntries(
			tenants.map((t) => [t.name, fixed_cost_per_person])
		);
		const absence_intervals = tenants.map((t) => ({
			name: t.name,
			intervals: t.absences.map((a) => ({
				start: parseISO(a.start),
				end: parseISO(a.end)
			}))
		}));

		for (let i = 0; i < total_days; i++) {
			const cur_date = addDays(start, i);
			const present_count =
				tenants.length -
				absence_intervals.filter((t) =>
					t.intervals.some((interval) => isWithinInterval(cur_date, interval))
				).length;

			// Everyone pays their share of idle costs
			const idleSplitPerPerson = idle_daily_rate / tenants.length;
			tenants.forEach((t) => {
				split[t.name] += idleSplitPerPerson;
			});

			// Only present people pay active costs
			const activeSplitPerPerson = active_daily_rate / present_count;
			absence_intervals.forEach((t) => {
				if (!t.intervals.some((interval) => isWithinInterval(cur_date, interval))) {
					split[t.name] += activeSplitPerPerson;
				}
			});
		}

		// apply GST
		Object.keys(split).forEach((name) => {
			split[name] *= 1 + gst_percent / 100;
		});

		return split;
	});

	let bill_summary: Summary = $derived.by(() => {
		const total_w_gst = Object.values(bill_split).reduce((a, b) => a + b, 0);
		const total_wo_gst = total_w_gst / (1 + gst_percent / 100);
		return { total_w_gst, total_wo_gst };
	});
</script>

<main class="container mx-auto max-w-3xl px-4 py-8 font-mono text-sm">
	<!-- Bill Details Section -->
	<section class="mb-8">
		<div class="mb-4 flex justify-between text-2xl font-bold">
			<h2>😳🔥</h2>
			<h2>🔥😳</h2>
		</div>
		<div class="grid grid-cols-1 gap-4 md:grid-cols-5">
			<div class="space-y-2">
				<label for="bill-start" class="block text-sm font-medium text-gray-700">Start Date</label>
				<input
					type="date"
					id="bill-start"
					bind:value={bill_start}
					class="w-full rounded-md border-gray-300 shadow-sm focus:border-blue-500 focus:ring-blue-500"
				/>
			</div>
			<div class="space-y-2">
				<label for="bill-end" class="block text-sm font-medium text-gray-700">End Date</label>
				<input
					type="date"
					id="bill-end"
					bind:value={bill_end}
					class="w-full rounded-md border-gray-300 shadow-sm focus:border-blue-500 focus:ring-blue-500"
				/>
			</div>
			<div class="space-y-2">
				<label for="bill-amount" class="block text-sm font-medium text-gray-700"
					>Variable Cost</label
				>
				<input
					type="number"
					id="bill-amount"
					bind:value={var_amt}
					class="w-full rounded-md border-gray-300 shadow-sm focus:border-blue-500 focus:ring-blue-500"
				/>
			</div>
			<div class="space-y-2">
				<label for="fixed-cost" class="block text-sm font-medium text-gray-700">Fixed Cost</label>
				<input
					type="number"
					id="fixed-cost"
					bind:value={fixed_amt}
					class="w-full rounded-md border-gray-300 shadow-sm focus:border-blue-500 focus:ring-blue-500"
				/>
			</div>
			<div class="space-y-2">
				<label for="gst" class="block text-sm font-medium text-gray-700">GST (%)</label>
				<input
					type="number"
					id="gst"
					bind:value={gst_percent}
					class="w-full rounded-md border-gray-300 shadow-sm focus:border-blue-500 focus:ring-blue-500"
				/>
			</div>
		</div>
	</section>

	<section class="mb-8">
		<div class="space-y-2">
			<label for="idle-cost" class="block text-sm font-medium text-gray-700">
				Idle Cost Percentage: {idle_cost_percent}%
			</label>
			<input
				type="range"
				id="idle-cost"
				bind:value={idle_cost_percent}
				min="0"
				max="100"
				class="w-full"
			/>
			<p class="text-sm text-gray-500">
				Percentage of the bill that will be split equally among everyone, even if they are not in 😭
			</p>
		</div>
	</section>

	<!-- Tenants Section -->
	<section class="mb-8">
		<h2 class="mb-4 text-2xl font-bold">Tenants</h2>
		{#each tenants as tenant, idx}
			<div class="mb-6 rounded-lg bg-gray-50 p-4">
				<div class="mb-4 flex items-center justify-between">
					<h3 class="text-xl font-semibold">{tenant.name}</h3>
					<button
						onclick={() => add_absence(idx)}
						class="rounded-md bg-blue-500 px-4 py-2 text-white hover:bg-blue-600 focus:outline-none focus:ring-2 focus:ring-blue-500 focus:ring-offset-2"
					>
						+ Absence
					</button>
				</div>

				<!-- Absences -->
				<div class="space-y-4">
					{#each tenant.absences as absence, absence_idx}
						<div class="flex items-center justify-between space-x-6">
							<div class="grid w-full grid-cols-1 gap-4 md:grid-cols-2">
								<div class="space-y-2">
									<label class="block text-sm font-medium text-gray-700">Absence Start</label>
									<input
										type="date"
										bind:value={absence.start}
										oninput={(e) =>
											update_absence(idx, absence_idx, 'start', e.currentTarget.value)}
										class="w-full rounded-md border-gray-300 shadow-sm focus:border-blue-500 focus:ring-blue-500"
									/>
								</div>
								<div class="space-y-2">
									<label class="block text-sm font-medium text-gray-700">Absence End</label>
									<input
										type="date"
										bind:value={absence.end}
										oninput={(e) => update_absence(idx, absence_idx, 'end', e.currentTarget.value)}
										class="w-full rounded-md border-gray-300 shadow-sm focus:border-blue-500 focus:ring-blue-500"
									/>
								</div>
							</div>
							<button
								onclick={() => {
									update_absence(idx, absence_idx, 'start', bill_start);
									update_absence(idx, absence_idx, 'end', bill_end);
								}}
								class="whitespace-nowrap rounded-md bg-gray-500 px-2 py-1 text-white hover:bg-gray-600 focus:outline-none focus:ring-2 focus:ring-gray-500 focus:ring-offset-2"
							>
								whole time
							</button>
						</div>
					{/each}
				</div>
			</div>
		{/each}
	</section>

	<!-- Results Section -->
	{#if Object.keys(bill_split).length > 0}
		<section class="mb-8">
			<h2 class="mb-4 text-2xl font-bold">Split</h2>
			<div class="overflow-hidden rounded-lg bg-white shadow">
				<div class="divide-y divide-gray-200">
					{#each Object.entries(bill_split) as [name, amount]}
						<div class="flex items-center justify-between px-4 py-3">
							<span class="font-medium">{name}</span>
							<span class="text-gray-900">${amount.toFixed(2)}</span>
						</div>
					{/each}
				</div>
			</div>

			<div class="flex items-center justify-between px-4 py-3">
				<span>Total (+GST)</span>
				<span>{bill_summary.total_w_gst.toFixed(2)}</span>
			</div>

			<div class="flex items-center justify-between px-4 py-3">
				<span>Total (w/o GST)</span>
				<span>{bill_summary.total_wo_gst.toFixed(2)}</span>
			</div>
		</section>
	{/if}
</main>
