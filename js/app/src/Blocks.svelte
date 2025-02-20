<script lang="ts">
	import { tick } from "svelte";
	import { _ } from "svelte-i18n";
	import { client } from "@gradio/client";

	import type { LoadingStatusCollection } from "./stores";

	import type { ComponentMeta, Dependency, LayoutNode } from "./types";
	import type { UpdateTransaction } from "./init";
	import { setupi18n } from "./i18n";
	import { ApiDocs } from "./api_docs/";
	import type { ThemeMode, Payload } from "./types";
	import { Toast } from "@gradio/statustracker";
	import type { ToastMessage } from "@gradio/statustracker";
	import type { ShareData } from "@gradio/utils";
	import MountComponents from "./MountComponents.svelte";

	import logo from "./images/logo.svg";
	import api_logo from "./api_docs/img/api-logo.svg";
	import { create_components, AsyncFunction } from "./init";

	setupi18n();

	export let root: string;
	export let components: ComponentMeta[];
	export let layout: LayoutNode;
	export let dependencies: Dependency[];
	export let title = "Gradio";
	export let analytics_enabled = false;
	export let target: HTMLElement;
	export let autoscroll: boolean;
	export let show_api = true;
	export let show_footer = true;
	export let control_page_title = false;
	export let app_mode: boolean;
	export let theme_mode: ThemeMode;
	export let app: Awaited<ReturnType<typeof client>>;
	export let space_id: string | null;
	export let version: string;
	export let js: string | null;
	export let fill_height = false;
	export let ready: boolean;

	const {
		layout: _layout,
		targets,
		update_value,
		get_data,
		loading_status,
		scheduled_updates,
		create_layout
	} = create_components();

	$: create_layout({
		components,
		layout,
		dependencies,
		root,
		app,
		options: {
			fill_height
		}
	});

	$: {
		ready = !!$_layout;
	}

	let params = new URLSearchParams(window.location.search);
	let api_docs_visible = params.get("view") === "api" && show_api;
	function set_api_docs_visible(visible: boolean): void {
		api_docs_visible = visible;
		let params = new URLSearchParams(window.location.search);
		if (visible) {
			params.set("view", "api");
		} else {
			params.delete("view");
		}
		history.replaceState(null, "", "?" + params.toString());
	}

	export let render_complete = false;
	async function handle_update(data: any, fn_index: number): Promise<void> {
		const outputs = dependencies[fn_index].outputs;

		const meta_updates = data?.map((value: any, i: number) => {
			return {
				id: outputs[i],
				prop: "value_is_output",
				value: true
			};
		});

		update_value(meta_updates);

		await tick();

		const updates: UpdateTransaction[] = [];

		data?.forEach((value: any, i: number) => {
			if (
				typeof value === "object" &&
				value !== null &&
				value.__type__ === "update"
			) {
				for (const [update_key, update_value] of Object.entries(value)) {
					if (update_key === "__type__") {
						continue;
					} else {
						updates.push({
							id: outputs[i],
							prop: update_key,
							value: update_value
						});
					}
				}
			} else {
				updates.push({
					id: outputs[i],
					prop: "value",
					value
				});
			}
		});
		update_value(updates);
	}

	let submit_map: Map<number, ReturnType<typeof app.submit>> = new Map();

	let handled_dependencies: number[][] = [];

	let messages: (ToastMessage & { fn_index: number })[] = [];
	function new_message(
		message: string,
		fn_index: number,
		type: ToastMessage["type"]
	): ToastMessage & { fn_index: number } {
		return {
			message,
			fn_index,
			type,
			id: ++_error_id
		};
	}

	let _error_id = -1;

	let user_left_page = false;
	document.addEventListener("visibilitychange", function () {
		if (document.visibilityState === "hidden") {
			user_left_page = true;
		}
	});

	const MESSAGE_QUOTE_RE = /^'([^]+)'$/;

	const DUPLICATE_MESSAGE = $_("blocks.long_requests_queue");
	const MOBILE_QUEUE_WARNING = $_("blocks.connection_can_break");
	const MOBILE_RECONNECT_MESSAGE = $_("blocks.lost_connection");
	const SHOW_DUPLICATE_MESSAGE_ON_ETA = 15;
	const SHOW_MOBILE_QUEUE_WARNING_ON_ETA = 10;
	const is_mobile_device =
		/Android|webOS|iPhone|iPad|iPod|BlackBerry|IEMobile|Opera Mini/i.test(
			navigator.userAgent
		);
	let showed_duplicate_message = false;
	let showed_mobile_warning = false;

	// as state updates are not synchronous, we need to ensure updates are flushed before triggering any requests
	function wait_then_trigger_api_call(
		dep_index: number,
		trigger_id: number | null = null,
		event_data: unknown = null
	): void {
		let _unsub = (): void => {};
		function unsub(): void {
			_unsub();
		}
		if ($scheduled_updates) {
			_unsub = scheduled_updates.subscribe((updating) => {
				if (!updating) {
					trigger_api_call(dep_index, trigger_id, event_data);
					unsub();
				}
			});
		} else {
			trigger_api_call(dep_index, trigger_id, event_data);
		}
	}

	async function trigger_api_call(
		dep_index: number,
		trigger_id: number | null = null,
		event_data: unknown = null
	): Promise<void> {
		let dep = dependencies[dep_index];

		const current_status = loading_status.get_status_for_fn(dep_index);
		messages = messages.filter(({ fn_index }) => fn_index !== dep_index);
		if (dep.cancels) {
			await Promise.all(
				dep.cancels.map(async (fn_index) => {
					const submission = submit_map.get(fn_index);
					submission?.cancel();
					return submission;
				})
			);
		}
		if (current_status === "pending" || current_status === "generating") {
			dep.pending_request = true;
		}

		let payload: Payload = {
			fn_index: dep_index,
			data: await Promise.all(dep.inputs.map((id) => get_data(id))),
			event_data: dep.collects_event_data ? event_data : null,
			trigger_id: trigger_id
		};

		if (dep.frontend_fn) {
			dep
				.frontend_fn(
					payload.data.concat(
						await Promise.all(dep.inputs.map((id) => get_data(id)))
					)
				)
				.then((v: unknown[]) => {
					if (dep.backend_fn) {
						payload.data = v;
						make_prediction(payload);
					} else {
						handle_update(v, dep_index);
					}
				});
		} else {
			if (dep.backend_fn) {
				if (dep.trigger_mode === "once") {
					if (!dep.pending_request) make_prediction(payload);
				} else if (dep.trigger_mode === "multiple") {
					make_prediction(payload);
				} else if (dep.trigger_mode === "always_last") {
					if (!dep.pending_request) {
						make_prediction(payload);
					} else {
						dep.final_event = payload;
					}
				}
			}
		}

		async function make_prediction(payload: Payload): Promise<void> {
			const submission = app
				.submit(
					payload.fn_index,
					payload.data as unknown[],
					payload.event_data,
					payload.trigger_id
				)
				.on("data", ({ data, fn_index }) => {
					if (dep.pending_request && dep.final_event) {
						dep.pending_request = false;
						make_prediction(dep.final_event);
					}
					dep.pending_request = false;
					handle_update(data, fn_index);
				})
				.on("status", ({ fn_index, ...status }) => {
					requestAnimationFrame(() => {
						//@ts-ignore
						loading_status.update({
							...status,
							status: status.stage,
							progress: status.progress_data,
							fn_index
						});
						if (
							!showed_duplicate_message &&
							space_id !== null &&
							status.position !== undefined &&
							status.position >= 2 &&
							status.eta !== undefined &&
							status.eta > SHOW_DUPLICATE_MESSAGE_ON_ETA
						) {
							showed_duplicate_message = true;
							messages = [
								new_message(DUPLICATE_MESSAGE, fn_index, "warning"),
								...messages
							];
						}
						if (
							!showed_mobile_warning &&
							is_mobile_device &&
							status.eta !== undefined &&
							status.eta > SHOW_MOBILE_QUEUE_WARNING_ON_ETA
						) {
							showed_mobile_warning = true;
							messages = [
								new_message(MOBILE_QUEUE_WARNING, fn_index, "warning"),
								...messages
							];
						}

						if (status.stage === "complete") {
							dependencies.map(async (dep, i) => {
								if (dep.trigger_after === fn_index) {
									wait_then_trigger_api_call(i, payload.trigger_id);
								}
							});

							submission.destroy();
						}
						if (status.broken && is_mobile_device && user_left_page) {
							window.setTimeout(() => {
								messages = [
									new_message(MOBILE_RECONNECT_MESSAGE, fn_index, "error"),
									...messages
								];
							}, 0);
							wait_then_trigger_api_call(
								dep_index,
								payload.trigger_id,
								event_data
							);
							user_left_page = false;
						} else if (status.stage === "error") {
							if (status.message) {
								const _message = status.message.replace(
									MESSAGE_QUOTE_RE,
									(_, b) => b
								);
								messages = [
									new_message(_message, fn_index, "error"),
									...messages
								];
							}
							dependencies.map(async (dep, i) => {
								if (
									dep.trigger_after === fn_index &&
									!dep.trigger_only_on_success
								) {
									wait_then_trigger_api_call(i, payload.trigger_id);
								}
							});

							submission.destroy();
						}
					});
				})
				.on("log", ({ log, fn_index, level }) => {
					messages = [new_message(log, fn_index, level), ...messages];
				});

			submit_map.set(dep_index, submission);
		}
	}

	function trigger_share(title: string | undefined, description: string): void {
		if (space_id === null) {
			return;
		}
		const discussion_url = new URL(
			`https://huggingface.co/spaces/${space_id}/discussions/new`
		);
		if (title !== undefined && title.length > 0) {
			discussion_url.searchParams.set("title", title);
		}
		discussion_url.searchParams.set("description", description);
		window.open(discussion_url.toString(), "_blank");
	}

	function handle_error_close(e: Event & { detail: number }): void {
		const _id = e.detail;
		messages = messages.filter((m) => m.id !== _id);
	}

	const is_external_url = (link: string | null): boolean =>
		!!(link && new URL(link, location.href).origin !== location.origin);

	async function handle_mount(): Promise<void> {
		if (js) {
			let blocks_frontend_fn = new AsyncFunction(
				`let result = await (${js})();
					return (!Array.isArray(result)) ? [result] : result;`
			);
			await blocks_frontend_fn();
		}

		await tick();

		var a = target.getElementsByTagName("a");

		for (var i = 0; i < a.length; i++) {
			const _target = a[i].getAttribute("target");
			const _link = a[i].getAttribute("href");

			// only target anchor tags with external links
			if (is_external_url(_link) && _target !== "_blank")
				a[i].setAttribute("target", "_blank");
		}

		// handle load triggers
		dependencies.forEach((dep, i) => {
			if (dep.targets[0][1] === "load") {
				wait_then_trigger_api_call(i);
			}
		});

		if (render_complete) return;

		target.addEventListener("prop_change", (e: Event) => {
			if (!isCustomEvent(e)) throw new Error("not a custom event");
			const { id, prop, value } = e.detail;
			update_value([{ id, prop, value }]);
		});
		target.addEventListener("gradio", (e: Event) => {
			if (!isCustomEvent(e)) throw new Error("not a custom event");

			const { id, event, data } = e.detail;

			if (event === "share") {
				const { title, description } = data as ShareData;
				trigger_share(title, description);
			} else if (event === "error" || event === "warning") {
				messages = [new_message(data, -1, event), ...messages];
			} else {
				const deps = $targets[id]?.[event];

				deps?.forEach((dep_id) => {
					wait_then_trigger_api_call(dep_id, id, data);
				});
			}
		});

		render_complete = true;
	}

	function handle_destroy(id: number): void {
		handled_dependencies = handled_dependencies.map((dep) => {
			return dep.filter((_id) => _id !== id);
		});
	}

	$: set_status($loading_status);

	function set_status(statuses: LoadingStatusCollection): void {
		const updates = Object.entries(statuses).map(([id, loading_status]) => {
			let dependency = dependencies[loading_status.fn_index];
			loading_status.scroll_to_output = dependency.scroll_to_output;
			loading_status.show_progress = dependency.show_progress;
			return {
				id: parseInt(id),
				prop: "loading_status",
				value: loading_status
			};
		});

		const inputs_to_update = loading_status.get_inputs_to_update();

		const additional_updates = Array.from(inputs_to_update).map(
			([id, pending_status]) => {
				return {
					id,
					prop: "pending",
					value: pending_status === "pending"
				};
			}
		);

		update_value([...updates, ...additional_updates]);
	}

	function isCustomEvent(event: Event): event is CustomEvent {
		return "detail" in event;
	}
</script>

<svelte:head>
	{#if control_page_title}
		<title>{title}</title>
	{/if}
	{#if analytics_enabled}
		<script
			async
			defer
			src="https://www.googletagmanager.com/gtag/js?id=UA-156449732-1"
		></script>
		<script>
			window.dataLayer = window.dataLayer || [];
			function gtag() {
				dataLayer.push(arguments);
			}
			gtag("js", new Date());
			gtag("config", "UA-156449732-1", {
				cookie_flags: "samesite=none;secure"
			});
		</script>
	{/if}
</svelte:head>

<div class="wrap" style:min-height={app_mode ? "100%" : "auto"}>
	<div class="contain" style:flex-grow={app_mode ? "1" : "auto"}>
		{#if $_layout}
			<MountComponents
				rootNode={$_layout}
				{root}
				{target}
				{theme_mode}
				on:mount={handle_mount}
				on:destroy={({ detail }) => handle_destroy(detail)}
				{version}
				{autoscroll}
			/>
		{/if}
	</div>

	{#if show_footer}
		<footer>
			{#if show_api}
				<button
					on:click={() => {
						set_api_docs_visible(!api_docs_visible);
					}}
					class="show-api"
				>
					{$_("errors.use_via_api")}
					<img src={api_logo} alt={$_("common.logo")} />
				</button>
				<div>·</div>
			{/if}
			<a
				href="https://gradio.app"
				class="built-with"
				target="_blank"
				rel="noreferrer"
			>
				{$_("common.built_with_gradio")}
				<img src={logo} alt={$_("common.logo")} />
			</a>
		</footer>
	{/if}
</div>

{#if api_docs_visible && $_layout}
	<div class="api-docs">
		<!-- TODO: fix -->
		<!-- svelte-ignore a11y-click-events-have-key-events-->
		<!-- svelte-ignore a11y-no-static-element-interactions-->
		<div
			class="backdrop"
			on:click={() => {
				set_api_docs_visible(false);
			}}
		/>
		<div class="api-docs-wrap">
			<ApiDocs
				root_node={$_layout}
				on:close={() => {
					set_api_docs_visible(false);
				}}
				{dependencies}
				{root}
				{app}
				{space_id}
			/>
		</div>
	</div>
{/if}

{#if messages}
	<Toast {messages} on:close={handle_error_close} />
{/if}

<style>
	.wrap {
		display: flex;
		flex-grow: 1;
		flex-direction: column;
		width: var(--size-full);
		font-weight: var(--body-text-weight);
		font-size: var(--body-text-size);
	}

	.contain {
		display: flex;
		flex-direction: column;
	}

	footer {
		display: flex;
		justify-content: center;
		margin-top: var(--size-4);
		color: var(--body-text-color-subdued);
	}

	footer > * + * {
		margin-left: var(--size-2);
	}

	.show-api {
		display: flex;
		align-items: center;
	}
	.show-api:hover {
		color: var(--body-text-color);
	}

	.show-api img {
		margin-right: var(--size-1);
		margin-left: var(--size-2);
		width: var(--size-3);
	}

	.built-with {
		display: flex;
		align-items: center;
	}

	.built-with:hover {
		color: var(--body-text-color);
	}

	.built-with img {
		margin-right: var(--size-1);
		margin-left: var(--size-1);
		margin-bottom: 1px;
		width: var(--size-4);
	}

	.api-docs {
		display: flex;
		position: fixed;
		top: 0;
		right: 0;
		z-index: var(--layer-5);
		background: rgba(0, 0, 0, 0.5);
		width: var(--size-screen);
		height: var(--size-screen-h);
	}

	.backdrop {
		flex: 1 1 0%;
		-webkit-backdrop-filter: blur(4px);
		backdrop-filter: blur(4px);
	}

	.api-docs-wrap {
		box-shadow: var(--shadow-drop-lg);
		background: var(--background-fill-primary);
		overflow-x: hidden;
		overflow-y: auto;
	}

	@media (--screen-md) {
		.api-docs-wrap {
			border-top-left-radius: var(--radius-lg);
			border-bottom-left-radius: var(--radius-lg);
			width: 950px;
		}
	}

	@media (--screen-xxl) {
		.api-docs-wrap {
			width: 1150px;
		}
	}
</style>
