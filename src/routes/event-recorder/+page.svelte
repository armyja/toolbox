<script lang="ts">
	import { onMount, tick } from 'svelte';
	import { format, intervalToDuration, type Duration, startOfDay, endOfDay } from 'date-fns';
	import { openDB, type DBSchema, type IDBPDatabase, type StoreValue } from 'idb';
	// 初始化变量
	let selected_date = new Date();
	// Database
	interface MyDB extends DBSchema {
		tags: {
			key: string;
			value: number;
		};
		events: {
			value: {
				id?: number;
				date: Date;
				tag: string;
				note: string;
			};
			key: number;
			indexes: { 'by-date': Date; 'by-tag': string };
		};
	}
	let database: IDBPDatabase<MyDB>;
	async function useDB() {
		// Opens the first version of the 'test-db1' database.
		// If the database does not exist, it will be created.
		const dbPromise = await openDB<MyDB>('my-db', 1, {
			upgrade(db) {
				const event_store = db.createObjectStore('events', {
					// The 'id' property of the object will be the key.
					keyPath: 'id',
					// If it isn't explicitly set, create a value by auto incrementing.
					autoIncrement: true
				});
				// Create an index on the 'date' property of the objects.
				event_store.createIndex('by-date', 'date');
				event_store.createIndex('by-tag', 'tag');
				db.createObjectStore('tags');
			}
		});
		database = dbPromise;
		return dbPromise;
	}
	async function add_tag(db: IDBPDatabase<MyDB>, tag: string) {
		await db.put('tags', 0, tag);
	}
	async function add_event(db: IDBPDatabase<MyDB>, val: StoreValue<MyDB, 'events'>) {
		await db.put('events', val);
	}
	async function init() {
		if (database === void 0) {
			database = await useDB();
			await tick();
		}
		notify();
	}
	async function notify() {
		fetch_data(selected_date);
	}
	// 插入数据
	let current_date: Date | undefined;
	let text = '';
	let note = '';
	function update_now() {
		current_date = new Date();
	}
	async function save_event() {
		let ret = text ? text.trim() : '';
		if (ret === '') {
			text = '';
			return;
		}
		await add_event(database, {
			date: current_date ? current_date : new Date(),
			tag: ret,
			note: note
		});
		current_date = undefined;
		text = '';
		note = '';
		notify();
	}
	function update_now_if_not_exist() {
		if (current_date === void 0) {
			current_date = new Date();
		}
	}
	// 展示数据
	interface Sevent extends StoreValue<MyDB, 'events'> {
		duration?: Duration;
	}
	let sRecords: Sevent[] = [];
	async function fetch_data(aDate: Date) {
		let records: StoreValue<MyDB, 'events'>[] = await database.getAllFromIndex(
			'events',
			'by-date',
			IDBKeyRange.bound(startOfDay(aDate), endOfDay(aDate))
		);
		let m_sRecords: Sevent[] = [];
		let map = new Map<string, Date>();
		for (let i of records) {
			let e: Sevent = {
				...i
			};
			if (map.get(i.tag) !== void 0) {
				e.duration = intervalToDuration({
					start: map.get(i.tag) || i.date,
					end: i.date
				});
			}
			map.set(i.tag, i.date);
			m_sRecords.unshift(e);
		}
		sRecords = m_sRecords;
	}
	async function find_date(is_after: boolean) {
		let bound_date = is_after ? endOfDay(selected_date) : startOfDay(selected_date);
		let range = is_after
			? IDBKeyRange.lowerBound(bound_date, true)
			: IDBKeyRange.upperBound(bound_date, true);
		let keys = await database.getAllKeysFromIndex('events', 'by-date', range);
		console.log("keys", keys)
		let index = is_after ? 0 : keys.length - 1;
		if (keys.length == 0) {
			return;
		}
		selected_date = (await database.get('events', keys[index]))?.date || new Date();
		notify()
	}
	let last_click = new Date();
	async function delete_events() {
		let d = new Date();
		if (d.getTime() - last_click.getTime() > 800) {
			last_click = new Date();
			return;
		}
		last_click = new Date();
		await Promise.allSettled(sRecords.map((e) => database.delete('events', e.id || 0)));
		notify();
	}
	async function delete_event(id: number) {
		await database.delete('events', id);
		notify();
	}
	function toZhCnDuration(duration: Duration) {
		let hStr = duration.hours ? duration.hours + ' ' + '时' : '';
		let mStr = duration.minutes ? duration.minutes + ' ' + '分' : '';
		let sStr = duration.seconds ? duration.seconds + ' ' + '秒' : '';
		return [hStr, mStr, sStr].join(' ');
	}
	onMount(() => {
		init();
	});
</script>

<svelte:head>
	<title>事件记录</title>
	<meta name="description" content="事件记录" />
</svelte:head>

<div class="text-column">
	<h1 class="text-xl">事件记录</h1>

	<p>自动生成时间戳；生成当日记录汇总。筛选日期；导出日志；清除事件。</p>

	<div class="flex justify-between">
		<div class="flex-none w-18 py-2">
			{current_date ? format(current_date, 'HH:mm:ss') : ''}
		</div>
		<div>
			T
			<input
				on:focus={update_now_if_not_exist}
				bind:value={text}
				type="text"
				maxlength="4"
				class="w-20 my-2 px-2 rounded-sm border-gray-300"
			/>
			N
			<input
				on:focus={update_now_if_not_exist}
				bind:value={note}
				type="text"
				maxlength="6"
				class="w-20 my-2 px-2 rounded-sm border-gray-300"
			/>
			<button on:click={update_now}>N</button>
			<button on:click={save_event}>S</button>
		</div>
	</div>
	<div class="flex justify-between">
		<div class="flex items-center">
			<button on:click={() => find_date(false)}>&lt;</button>
			<p class="mx-1 inline-block align-middle">{format(selected_date, 'yyyy-MM-dd')}</p>
			<button
				on:click={() => find_date(true)}
				disabled={new Date().toDateString() === selected_date.toDateString()}>&gt;</button
			>
		</div>
		<button on:click={delete_events}>X</button>
	</div>
	<ul>
		{#each sRecords as record}
			<li class="py-1">
				<span class="font-mono">{format(record.date, 'HH:mm:ss')}</span> - <span>{record.tag}</span>
				<span>{record.note ? '| ' + record.note : ''}</span>
				<span>{record.duration ? '| ' + toZhCnDuration(record.duration || {}) : ''}</span>
				<button on:click={() => delete_event(record.id || 0)}>X</button>
			</li>
		{/each}
	</ul>
</div>

<style lang="postcss">
	button {
		@apply rounded-md bg-slate-500 px-2 py-1 text-sm font-semibold text-white select-none disabled:bg-slate-400;
	}
</style>
