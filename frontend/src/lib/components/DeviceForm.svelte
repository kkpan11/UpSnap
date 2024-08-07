<script lang="ts">
	import { goto } from '$app/navigation';
	import DeviceFormPort from '$lib/components/DeviceFormPort.svelte';
	import LL from '$lib/i18n/i18n-svelte';
	import { permission, pocketbase } from '$lib/stores/pocketbase';
	import type { Device, Group, Port } from '$lib/types/device';
	import { faSave, faTrash, faX } from '@fortawesome/free-solid-svg-icons';
	import { onMount } from 'svelte';
	import Fa from 'svelte-fa';
	import toast from 'svelte-french-toast';

	export let device: Device;

	let deleteModal: HTMLDialogElement;
	let deviceGroups = [] as Group[];
	let newGroup = '';

	onMount(async () => {
		await getGroups();
	});

	async function save() {
		// create/update all ports
		let portIds: string[] = [];
		await Promise.all(
			device.expand.ports.map(async (port: Port) => {
				if (port.id === undefined) {
					const data = await createPort(port as Port);
					if (data === undefined) return;
					portIds = [...portIds, data.id];
				} else {
					const data = await updatePort(port as Port);
					if (data === undefined) return;
					portIds = [...portIds, data.id];
				}
			})
		);
		device.ip = device.ip.replaceAll(' ', '');
		device.mac = device.mac.replaceAll(' ', '');
		device.netmask = device.netmask.replaceAll(' ', '');
		device.ports = portIds;
		device.id ? updateDevice(device) : createDevice(device);
	}

	async function updateDevice(device: Device) {
		$pocketbase
			.collection('devices')
			.update(device.id, device)
			.then(() => {
				toast.success($LL.toasts.device_updated({ device: device.name }));
				goto('/');
			})
			.catch((err) => {
				toast.error(err.message);
			});
	}

	async function createDevice(device: Device) {
		device.created_by = $pocketbase.authStore.isAdmin
			? ''
			: ($pocketbase.authStore.model?.id ?? '');
		$pocketbase
			.collection('devices')
			.create(device)
			.then(() => {
				toast.success($LL.toasts.device_created({ device: device.name }));
				goto('/');
			})
			.catch((err) => {
				toast.error(err.message);
			});
	}

	async function createPort(port: Port): Promise<Port | undefined> {
		let newport: Port | undefined;
		await $pocketbase
			.collection('ports')
			.create(port)
			.then((data) => {
				newport = data as Port;
			})
			.catch((err) => {
				toast.error(err.message);
				newport = undefined;
			});
		return newport;
	}

	async function updatePort(port: Port): Promise<Port | undefined> {
		let newport: Port | undefined;
		await $pocketbase
			.collection('ports')
			.update(port.id, port)
			.then((data) => {
				newport = data as Port;
			})
			.catch((err) => {
				toast.error(err.message);
				newport = undefined;
			});
		return newport;
	}

	function deleteDevice() {
		$pocketbase
			.collection('devices')
			.delete(device.id)
			.then(() => {
				toast.success($LL.toasts.device_deleted({ device: device.name }));
				goto('/');
			})
			.catch((err) => {
				toast.error(err.message);
			});
	}

	function createEmptyPort() {
		if (device === undefined) return;
		device.expand.ports = [...device.expand.ports, { name: '', number: 1 } as Port];
	}

	async function getGroups() {
		$pocketbase
			.collection('device_groups')
			.getFullList()
			.then((res) => {
				deviceGroups = res as Group[];
			});
	}

	function addGroup() {
		if (!newGroup) return;
		$pocketbase
			.collection('device_groups')
			.create({
				name: newGroup
			})
			.then((res) => {
				deviceGroups = [...deviceGroups, res as Group];
				toast.success($LL.toasts.group_created({ group: newGroup }));
			})
			.catch((err) => {
				toast.error(err.message);
			});
	}

	function deleteGroup(group: Group) {
		const i = device.groups.indexOf(group.id);
		if (i !== -1) {
			device.groups.splice(i, 1);
			device.groups = device.groups;
		}
		$pocketbase
			.collection('device_groups')
			.delete(group.id)
			.then(async () => {
				await getGroups();
				toast.success($LL.toasts.group_deleted({ group: group.name }));
			})
			.catch((err) => {
				toast.error(err.message);
			});
	}

	function toggleGroup(id: string) {
		const i = device.groups.indexOf(id);
		if (i !== -1) {
			device.groups.splice(i, 1);
			device.groups = device.groups;
		} else {
			device.groups = [...device.groups, id];
		}
	}
</script>

<form on:submit|preventDefault={save}>
	<div class="card w-full bg-base-300 shadow-xl">
		<div class="card-body">
			<h2 class="card-title">{$LL.device.general()}</h2>
			<div class="grid grid-cols-1 gap-4 md:grid-cols-2 xl:grid-cols-4">
				<div class="form-control w-full max-w-xs">
					<label class="label" for="device-name">
						<div class="label-text">
							<span>{$LL.device.general_name()}</span>
							<span class="text-error">*</span>
						</div>
					</label>
					<input
						id="device-name"
						type="text"
						placeholder={$LL.device.general_name_placeholder()}
						class="input w-full max-w-xs"
						bind:value={device.name}
						required
					/>
				</div>
				<div class="form-control w-full max-w-xs">
					<label class="label" for="device-ip">
						<div class="label-text">
							<span>{$LL.device.general_ip()}</span>
							<span class="text-error">*</span>
						</div>
					</label>
					<input
						id="device-ip"
						type="text"
						placeholder="192.168.0.5"
						class="input w-full max-w-xs"
						bind:value={device.ip}
						required
					/>
				</div>
				<div class="form-control w-full max-w-xs">
					<label class="label" for="device-mac">
						<div class="label-text">
							<span>{$LL.device.general_mac()}</span>
							<span class="text-error">*</span>
						</div>
					</label>
					<input
						id="device-mac"
						type="text"
						placeholder="aa:bb:cc:dd:ee:ff"
						class="input w-full max-w-xs"
						bind:value={device.mac}
						required
					/>
				</div>
				<div class="form-control w-full max-w-xs">
					<label class="label" for="device-netmask">
						<div class="label-text">
							<span>{$LL.device.general_netmask()}</span>
							<span class="text-error">*</span>
						</div>
					</label>
					<input
						id="device-netmask"
						type="text"
						placeholder="255.255.255.0"
						class="input w-full max-w-xs"
						bind:value={device.netmask}
						required
					/>
				</div>
				<span class="badge self-center text-error">* {$LL.device.general_required_field()}</span>
			</div>
		</div>
	</div>
	<div class="card mt-6 w-full bg-base-300 shadow-xl">
		<div class="card-body">
			<h2 class="card-title">{$LL.device.ports()}</h2>
			<p class="my-2">{$LL.device.ports_desc()}</p>
			<div class="form-control w-full">
				<div class="grid grid-cols-1 gap-4 md:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4">
					<!-- eslint-disable-next-line @typescript-eslint/no-unused-vars -->
					{#each device.expand.ports as _, index}
						<DeviceFormPort bind:device {index} />
					{/each}
				</div>
				<button
					class="btn btn-primary mt-4 max-w-xs"
					on:click={() => createEmptyPort()}
					type="button">{$LL.device.ports_add_new()}</button
				>
			</div>
		</div>
	</div>
	<div class="card mt-6 w-full bg-base-300 shadow-xl">
		<div class="card-body">
			<h2 class="card-title">{$LL.device.link()}</h2>
			<p class="my-2">
				{$LL.device.link_desc()}
			</p>
			<div class="form-control w-full">
				<input
					type="url"
					placeholder="https:// ..."
					class="input w-full max-w-xs"
					bind:value={device.link}
				/>
			</div>
		</div>
	</div>
	<div class="card mt-6 w-full bg-base-300 shadow-xl">
		<div class="card-body">
			<h2 class="card-title">{$LL.device.ping()}</h2>
			<p class="my-2">
				<!-- eslint-disable svelte/no-at-html-tags -->
				{@html $LL.device.ping_desc()}
			</p>
			<div class="form-control w-full">
				<label class="label cursor-pointer" for="ping-cmd">
					<span class="label-text">{$LL.device.ping_cmd()}</span>
				</label>
				<input
					id="ping-cmd"
					type="text"
					placeholder="$:"
					class="input mb-2 w-full max-w-xs"
					bind:value={device.ping_cmd}
				/>
			</div>
		</div>
	</div>
	<div class="card mt-6 w-full bg-base-300 shadow-xl">
		<div class="card-body">
			<h2 class="card-title">{$LL.device.wake()}</h2>
			<p class="my-2">
				{$LL.device.wake_desc()}
				<!-- eslint-disable svelte/no-at-html-tags -->
				{@html $LL.settings.ping_interval_desc2()}
			</p>
			<div class="form-control w-full">
				<label class="label cursor-pointer" for="wake-cmd">
					<span class="label-text">{$LL.device.wake_cmd()}</span>
				</label>
				<input
					id="wake-cmd"
					type="text"
					placeholder="$:"
					class="input mb-2 w-full max-w-xs"
					bind:value={device.wake_cmd}
				/>
			</div>
			<div class="form-control flex flex-row flex-wrap gap-4">
				<div class="flex flex-row items-center gap-2">
					<input
						id="wake-confirm"
						type="checkbox"
						class="toggle toggle-success"
						bind:checked={device.wake_confirm}
					/>
					<label class="label cursor-pointer" for="wake-confirm">
						<span class="label-text">{$LL.device.require_confirmation()}</span>
					</label>
				</div>
			</div>
			<div class="form-control flex flex-row flex-wrap gap-4">
				<div class="w-full max-w-xs">
					<label class="label" for="wake-cron">
						<span class="label-text"
							>{$LL.device.wake_cron()}
							{#if device.wake_cron_enabled}
								<span class="text-error">*</span>
							{/if}
						</span>
					</label>
					<input
						id="wake-cron"
						type="text"
						placeholder="M H DoM M DoW"
						class="input w-full max-w-xs"
						bind:value={device.wake_cron}
						disabled={!device.wake_cron_enabled}
						required={device.wake_cron_enabled}
					/>
				</div>
				<div class="flex flex-col">
					<label class="label cursor-pointer" for="wake-cron-enable">
						<span class="label-text">{$LL.device.wake_cron_enable()}</span>
					</label>
					<input
						id="wake-cron-enable"
						type="checkbox"
						class="toggle toggle-success"
						bind:checked={device.wake_cron_enabled}
					/>
				</div>
			</div>
		</div>
	</div>

	<div class="card mt-6 w-full bg-base-300 shadow-xl">
		<div class="card-body">
			<h2 class="card-title">Sleep-On-LAN</h2>
			<p class="mt-2">
				<!-- eslint-disable svelte/no-at-html-tags -->
				{@html $LL.device.sol_desc1()}
			</p>
			<p>
				{$LL.device.sol_desc2()}
			</p>
			<p class="font-bold">
				<!-- eslint-disable svelte/no-at-html-tags -->
				{@html $LL.device.sol_desc3()}
			</p>
			<div class="mt-4 flex flex-row flex-wrap items-end gap-4">
				<div>
					<div class="form-control flex flex-row flex-wrap gap-4">
						<div class="flex flex-row items-center gap-2">
							<input
								id="sol-enable"
								type="checkbox"
								class="toggle toggle-success"
								bind:checked={device.sol_enabled}
							/>
							<label class="label cursor-pointer" for="sol-enable">
								<span class="label-text">{$LL.device.sol_enable()}</span>
							</label>
						</div>
					</div>
					<div class="form-control flex flex-col">
						<label class="label" for="sol-port">
							<span class="label-text"
								>{$LL.device.sol_port()}
								{#if device.sol_enabled}
									<span class="text-error">*</span>
								{/if}
							</span>
						</label>
						<input
							id="sol-port"
							type="number"
							min="1"
							max="65535"
							class="input"
							bind:value={device.sol_port}
							disabled={!device.sol_enabled}
							required={device.sol_enabled}
						/>
					</div>
				</div>
				{#if device.sol_enabled}
					<div>
						<div class="form-control flex flex-row flex-wrap gap-4">
							<div class="flex flex-row items-center gap-2">
								<input
									id="sol-auth"
									type="checkbox"
									class="toggle toggle-success"
									bind:checked={device.sol_auth}
								/>
								<label class="label cursor-pointer" for="sol-auth">
									<span class="label-text">{$LL.device.sol_authorization()}</span>
								</label>
							</div>
						</div>
						<div class="form-control flex flex-col">
							<label class="label" for="sol-user">
								<span class="label-text"
									>{$LL.device.sol_user()}
									{#if device.sol_auth}
										<span class="text-error">*</span>
									{/if}
								</span>
							</label>
							<input
								id="sol-user"
								type="text"
								placeholder={$LL.device.sol_user()}
								class="input"
								bind:value={device.sol_user}
								disabled={!device.sol_auth}
								required={device.sol_auth}
							/>
						</div>
					</div>
					<div class="form-control flex flex-col">
						<label class="label" for="sol-password">
							<span class="label-text"
								>{$LL.device.sol_password()}
								{#if device.sol_auth}
									<span class="text-error">*</span>
								{/if}
							</span>
						</label>
						<input
							id="sol-password"
							type="password"
							placeholder={$LL.device.sol_password()}
							class="input"
							bind:value={device.sol_password}
							disabled={!device.sol_auth}
							required={device.sol_auth}
						/>
					</div>
				{/if}
			</div>
		</div>
	</div>

	<div class="card mt-6 w-full bg-base-300 shadow-xl">
		<div class="card-body">
			<h2 class="card-title">{$LL.device.shutdown()}</h2>
			<p class="my-2">
				<!-- eslint-disable svelte/no-at-html-tags -->
				{@html $LL.device.shutdown_desc()}
			</p>
			<p class="my-2 font-bold">{$LL.device.shutdown_examples()}</p>
			<div class="mockup-code min-w-0 max-w-fit text-sm">
				<pre data-prefix="#"><code>{$LL.device.shutdown_examples_windows()}</code></pre>
				<pre data-prefix="$" class="text-warning"><code
						>net rpc shutdown -I 192.168.1.13 -U "user%password"</code
					></pre>
			</div>
			<div class="mockup-code min-w-0 max-w-fit text-sm">
				<pre data-prefix="#"><code>{$LL.device.shutdown_examples_linux()}</code></pre>
				<pre data-prefix="$" class="text-warning"><code
						>sshpass -p password ssh -o "StrictHostKeyChecking=no" user@192.168.1.13 "sudo poweroff"</code
					></pre>
			</div>
			<div class="form-control w-full">
				<label class="label cursor-pointer" for="shutdown-cmd">
					<span class="label-text">{$LL.device.shutdown_cmd()}</span>
				</label>
				<input
					id="shutdown-cmd"
					type="text"
					placeholder="$:"
					class="input mb-2 w-full max-w-xs"
					bind:value={device.shutdown_cmd}
				/>
			</div>
			<div class="form-control flex flex-row flex-wrap gap-4">
				<div class="flex flex-row items-center gap-2">
					<input
						id="shutdown-confirm"
						type="checkbox"
						class="toggle toggle-success"
						bind:checked={device.shutdown_confirm}
					/>
					<label class="label cursor-pointer" for="shutdown-confirm">
						<span class="label-text">{$LL.device.require_confirmation()}</span>
					</label>
				</div>
			</div>
			<p class="my-2">
				{$LL.device.shutdown_cron_desc()}
			</p>
			<div class="form-control flex flex-row flex-wrap gap-4">
				<div class="w-full max-w-xs">
					<label class="label" for="shutdown-cron">
						<span class="label-text"
							>{$LL.device.shutdown_cron()}
							{#if device.shutdown_cron_enabled}
								<span class="text-error">*</span>
							{/if}
						</span>
					</label>
					<input
						id="shutdown-cron"
						type="text"
						placeholder="M H DoM M DoW"
						class="input w-full max-w-xs"
						bind:value={device.shutdown_cron}
						disabled={!device.shutdown_cron_enabled}
						required={device.shutdown_cron_enabled}
					/>
				</div>
				<div class="flex flex-col">
					<label class="label cursor-pointer" for="shutdown-cron-enable">
						<span class="label-text">{$LL.device.shutdown_cron_enable()}</span>
					</label>
					<input
						id="shutdown-cron-enable"
						type="checkbox"
						class="toggle toggle-success"
						bind:checked={device.shutdown_cron_enabled}
					/>
				</div>
			</div>
		</div>
	</div>
	<div class="card mt-6 w-full bg-base-300 shadow-xl">
		<div class="card-body">
			<h2 class="card-title">{$LL.device.password()}</h2>
			<p class="my-2">
				<!-- eslint-disable svelte/no-at-html-tags -->
				{@html $LL.device.password_desc()}
			</p>
			<div class="form-control w-full">
				<input
					type="password"
					placeholder="123456"
					class="input w-full max-w-xs"
					maxlength="6"
					bind:value={device.password}
				/>
			</div>
		</div>
	</div>
	<div class="card mt-6 w-full bg-base-300 shadow-xl">
		<div class="card-body">
			<h2 class="card-title">{$LL.device.groups()}</h2>
			<p class="my-2">
				{$LL.device.groups_desc()}
			</p>
			<div class="flex flex-row flex-wrap gap-2">
				{#each deviceGroups as group}
					<div class="join">
						<div class=" tooltip" data-tip="Delete">
							<button
								class="btn btn-error join-item"
								type="button"
								on:click={() => deleteGroup(group)}><Fa icon={faX} /></button
							>
						</div>
						<div
							class="btn join-item bg-base-100 hover:bg-base-200"
							on:click={() => toggleGroup(group.id)}
							role="none"
						>
							<input
								type="checkbox"
								class="checkbox checked:checkbox-primary"
								checked={device.groups.indexOf(group.id) !== -1}
							/>
							{group.name}
						</div>
					</div>
				{/each}
			</div>
			<div class="join max-w-xs">
				<input
					class="input join-item input-bordered w-full"
					placeholder={$LL.device.groups_placeholder()}
					type="text"
					bind:value={newGroup}
				/>
				<button class="btn btn-primary join-item" type="button" on:click={() => addGroup()}
					>{$LL.buttons.add()}</button
				>
			</div>
		</div>
	</div>
	<div class="card-actions mt-6 justify-end gap-4">
		{#if $pocketbase.authStore.isAdmin || $permission.delete?.includes(device.id)}
			<button class="btn btn-error" type="button" on:click={() => deleteModal.showModal()}
				><Fa icon={faTrash} />{$LL.buttons.delete()}</button
			>
			<dialog class="modal" bind:this={deleteModal}>
				<form method="dialog" class="modal-box">
					<h3 class="text-lg font-bold">{$LL.users.confirm_delete_title()}</h3>
					<p class="py-4">{$LL.users.confirm_delete_desc({ username: device.name })}</p>
					<div class="modal-action">
						<button class="btn">{$LL.buttons.cancel()}</button>
						<button class="btn btn-error" on:click={() => deleteDevice()}
							>{$LL.buttons.delete()}</button
						>
					</div>
				</form>
			</dialog>
		{/if}
		<button class="btn btn-success" type="submit"><Fa icon={faSave} />{$LL.buttons.save()}</button>
	</div>
</form>
