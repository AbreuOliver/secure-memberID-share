<script lang="ts">
	import { onMount } from 'svelte';
	import { supabase } from '$lib/supabase';

	type Step = 'email' | 'code' | 'success';

	let step: Step = 'email';

	let email = '';
	let code = '';

	let sending = false;
	let verifying = false;

	let emailError: string | null = null;
	let codeError: string | null = null;
	let generalError: string | null = null;

	// Dynamic list of identifiers (all columns except email)
	let identifiers: { key: string; value: string }[] = [];

	// Track which identifier was just copied
	let copiedKey: string | null = null;
	let copyTimeout: ReturnType<typeof setTimeout> | null = null;

	// ---------- ADMIN LOGIC ----------
	const ADMIN_EMAILS = [
		'oliverabreu@icloud.com',
		'katielawhun@raleighchristian.com',
		'pennyhowell@raleighchristian.com'
	];

	let currentUserEmail: string | null = null;
	let isAdmin = false;

	let showAdmin = false;
	let adminRows: Record<string, any>[] = [];
	let adminColumns: string[] = [];
	let adminLoading = false;
	let adminError: string | null = null;

	function computeIsAdmin(emailVal: string | null) {
		if (!emailVal) return false;
		return ADMIN_EMAILS.includes(emailVal.toLowerCase());
	}

	// Restore session on page load
	onMount(async () => {
		try {
			const { data, error } = await supabase.auth.getUser();
			if (error || !data.user) return;

			currentUserEmail = data.user.email ?? null;
			isAdmin = computeIsAdmin(currentUserEmail);

			const { data: row, error: rowError } = await supabase
				.from('approved_user_info')
				.select('*')
				.eq('email', currentUserEmail)
				.single();

			if (rowError || !row) return;

			identifiers = Object.entries(row)
				.filter(([key, value]) => key !== 'email' && value)
				.map(([key, value]) => ({
					key: key.replace(/_/g, ' '),
					value: String(value)
				}));

			if (identifiers.length === 0) return;

			step = 'success';
		} catch (err) {
			console.error('Error restoring session:', err);
		}
	});

	function resetAll() {
		step = 'email';
		email = '';
		code = '';
		sending = false;
		verifying = false;
		emailError = null;
		codeError = null;
		generalError = null;
		identifiers = [];
		copiedKey = null;
		showAdmin = false;
	}

	function validateEmail(value: string): boolean {
		const trimmed = value.trim();
		if (!trimmed) {
			emailError = 'Please enter your email address.';
			return false;
		}
		const pattern = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
		if (!pattern.test(trimmed)) {
			emailError = 'Please enter a valid email address.';
			return false;
		}
		emailError = null;
		return true;
	}

	function validateCode(value: string): boolean {
		const trimmed = value.trim();
		if (!trimmed) {
			codeError = 'Please enter the 6-digit code.';
			return false;
		}
		if (!/^\d{6}$/.test(trimmed)) {
			codeError = 'The code must be exactly 6 digits.';
			return false;
		}
		codeError = null;
		return true;
	}

	async function handleEmailSubmit() {
		generalError = null;
		if (!validateEmail(email)) return;

		sending = true;

		try {
			const { error } = await supabase.auth.signInWithOtp({
				email: email.trim()
			});

			if (error) {
				console.error(error);
				generalError = error.message ?? 'Unable to send code. Please try again.';
				return;
			}

			step = 'code';
		} catch (err) {
			console.error(err);
			generalError = 'Something went wrong while sending the code.';
		} finally {
			sending = false;
		}
	}

	async function handleCodeSubmit() {
		generalError = null;
		if (!validateCode(code)) return;

		verifying = true;

		try {
			const { error: verifyError } = await supabase.auth.verifyOtp({
				email: email.trim(),
				token: code.trim(),
				type: 'email'
			});

			if (verifyError) {
				console.error(verifyError);
				generalError = verifyError.message ?? 'Invalid or expired code.';
				return;
			}

			// refresh user + admin status
			const { data: userData } = await supabase.auth.getUser();
			currentUserEmail = userData?.user?.email ?? null;
			isAdmin = computeIsAdmin(currentUserEmail);

			const { data: row, error: rowError } = await supabase
				.from('approved_user_info')
				.select('*')
				.eq('email', currentUserEmail)
				.single();

			if (rowError || !row) {
				console.error(rowError);
				generalError = 'No insurance info found for this email.';
				return;
			}

			identifiers = Object.entries(row)
				.filter(([key, value]) => key !== 'email' && value)
				.map(([key, value]) => ({
					key: key.replace(/_/g, ' '),
					value: String(value)
				}));

			if (identifiers.length === 0) {
				generalError = 'No identifiers available for this user.';
				return;
			}

			copiedKey = null;
			step = 'success';
		} catch (err) {
			console.error(err);
			generalError = 'Something went wrong during verification.';
		} finally {
			verifying = false;
		}
	}

	async function copyIdentifier(key: string, value: string) {
		try {
			await navigator.clipboard.writeText(value);

			if (copyTimeout) {
				clearTimeout(copyTimeout);
			}

			copiedKey = key;

			copyTimeout = setTimeout(() => {
				copiedKey = null;
			}, 2000);
		} catch (err) {
			console.error('Failed to copy identifier:', err);
		}
	}

	async function handleSignOut() {
		try {
			await supabase.auth.signOut();
		} catch (err) {
			console.error('Error signing out:', err);
		} finally {
			resetAll();
		}
	}

	// ---------- ADMIN PANEL FUNCTIONS ----------

	async function openAdmin() {
		adminError = null;
		adminLoading = true;
		showAdmin = true;

		try {
			const { data, error } = await supabase
				.from('approved_user_info')
				.select('*')
				.order('email', { ascending: true });

			if (error) {
				console.error(error);
				adminError = 'Failed to load admin data.';
				return;
			}

			adminRows = data ?? [];

			// Derive editable columns from first row (everything except email)
			const first = adminRows[0];
			adminColumns = first ? Object.keys(first).filter((key) => key !== 'email') : [];
		} catch (err) {
			console.error(err);
			adminError = 'Unexpected error while loading admin data.';
		} finally {
			adminLoading = false;
		}
	}

	function addAdminRow() {
		const base: Record<string, any> = { email: '' };
		for (const col of adminColumns) {
			base[col] = '';
		}
		adminRows = [...adminRows, base];
	}

	async function saveAdmin() {
		adminError = null;
		adminLoading = true;

		try {
			const { error } = await supabase
				.from('approved_user_info')
				.upsert(adminRows, { onConflict: 'email' });

			if (error) {
				console.error(error);
				adminError = 'Failed to save changes.';
				return;
			}
		} catch (err) {
			console.error(err);
			adminError = 'Unexpected error while saving.';
		} finally {
			adminLoading = false;
		}
	}

	async function toggleAdminView() {
		if (!showAdmin) {
			// first time / switching to admin: load data
			await openAdmin();
		} else {
			// going back to member IDs
			showAdmin = false;
		}
	}

	// --- Dynamic Color Logic ---
	// The color class string reacts whenever showAdmin changes.
	$: buttonColorClasses = showAdmin
		? 'bg-green-600 hover:bg-green-700' // Green when showing 'Back to Member IDs' (showAdmin is true)
		: 'bg-blue-600 hover:bg-blue-700'; // Blue when showing 'View Admin Table' (showAdmin is false)
	// ---------------------------
</script>

<div class="flex min-h-screen w-full items-center justify-center bg-white p-6 text-neutral-900">
	<div class="w-full max-w-screen space-y-8">
		{#if step !== 'success'}
			<!-- Landing / form view -->
			<header class="space-y-4 text-center">
				<h1 class="text-4xl font-semibold tracking-tight">Access Your Insurance Info</h1>
				<p class="text-lg text-neutral-700">
					Enter your RCA/BBC email address to receive a one-time code and securely view your
					insurance identifiers.
				</p>
			</header>

			{#if generalError}
				<div
					class="rounded-lg border border-red-200 bg-red-50 px-4 py-3 text-center text-sm text-red-700"
				>
					{generalError}
				</div>
			{/if}

			{#if step === 'email'}
				<section class="mx-auto max-w-xl">
					<form class="space-y-4" on:submit|preventDefault={handleEmailSubmit}>
						<div class="space-y-2">
							<label for="email" class="block text-sm font-medium text-neutral-900">
								RCA/BBC email address
							</label>
							<input
								id="email"
								type="email"
								bind:value={email}
								class="block w-full rounded-lg border border-neutral-300 bg-white px-3.5 py-2.5 text-base text-neutral-900 shadow-sm placeholder:text-neutral-400 focus-visible:ring-2 focus-visible:ring-[#8D2417] focus-visible:ring-offset-1 focus-visible:ring-offset-white focus-visible:outline-none"
								autocomplete="email"
							/>
							{#if emailError}
								<p class="mt-1 text-xs text-red-600">{emailError}</p>
							{/if}
						</div>

						<button
							type="submit"
							class="inline-flex w-full items-center justify-center rounded-xl bg-[#8D2417] px-4 py-3 text-lg font-semibold text-white shadow-sm transition hover:bg-[#a02a1a] focus-visible:ring-2 focus-visible:ring-[#8D2417] focus-visible:ring-offset-2 focus-visible:ring-offset-white focus-visible:outline-none disabled:cursor-not-allowed disabled:opacity-60"
							disabled={sending}
						>
							{sending ? 'Sending code…' : 'Send code'}
						</button>
					</form>
				</section>
			{/if}

			{#if step === 'code'}
				<section class="mx-auto max-w-xl">
					<form class="space-y-4" on:submit|preventDefault={handleCodeSubmit}>
						<div class="space-y-2">
							<label for="code" class="block text-sm font-medium text-neutral-900">
								6-digit code
							</label>
							<input
								id="code"
								type="text"
								inputmode="numeric"
								maxlength="6"
								placeholder="••••••"
								bind:value={code}
								class="block w-full rounded-lg border border-neutral-300 bg-white px-3.5 py-2.5 text-center text-lg tracking-[0.35em] text-neutral-900 tabular-nums shadow-sm placeholder:text-neutral-400 focus-visible:ring-2 focus-visible:ring-[#8D2417] focus-visible:ring-offset-1 focus-visible:ring-offset-white focus-visible:outline-none"
								autocomplete="one-time-code"
							/>
							{#if codeError}
								<p class="mt-1 text-xs text-red-600">{codeError}</p>
							{/if}
						</div>

						<button
							type="submit"
							class="inline-flex w-full items-center justify-center rounded-xl bg-[#8D2417] px-4 py-3 text-lg font-semibold text-white shadow-sm transition hover:bg-[#a02a1a] focus-visible:ring-2 focus-visible:ring-[#8D2417] focus-visible:ring-offset-2 focus-visible:ring-offset-white focus-visible:outline-none disabled:cursor-not-allowed disabled:opacity-60"
							disabled={verifying}
						>
							{verifying ? 'Verifying…' : 'Verify code'}
						</button>

						<button
							type="button"
							class="w-full text-center text-sm text-neutral-600 underline underline-offset-2 hover:text-neutral-900"
							on:click={resetAll}
						>
							Use a different email or start over
						</button>
					</form>
				</section>
			{/if}
		{:else}
			<!-- SUCCESS: identifiers replace the landing content -->
			<header class="space-y-3 text-center">
				<h1 class="text-3xl font-semibold tracking-tight">
					{#if isAdmin && showAdmin}
						Admin: Approved User Info
					{:else}
						Your Insurance Identifiers
					{/if}
				</h1>
				<p class="text-base text-neutral-700">
					{#if isAdmin && showAdmin}
						Manage the list of approved users and their identifiers.
					{:else}
						These are the identifiers associated with your account. You can copy each one
						individually.
					{/if}
				</p>
			</header>

			{#if generalError}
				<div class="rounded-lg border border-red-200 bg-red-50 px-4 py-3 text-sm text-red-700">
					{generalError}
				</div>
			{/if}

			{#if isAdmin && showAdmin}
				<!-- ADMIN VIEW INSTEAD OF MEMBER IDs -->
				<section class="mt-6 rounded-xl border border-neutral-200 bg-neutral-50 p-4">
					<h2 class="mb-3 text-lg font-semibold">Approved User Info</h2>

					{#if adminError}
						<div
							class="mb-3 rounded-md border border-red-200 bg-red-50 px-3 py-2 text-sm text-red-700"
						>
							{adminError}
						</div>
					{/if}

					{#if adminLoading}
						<p class="text-sm text-neutral-600">Loading…</p>
					{:else if adminRows.length === 0}
						<p class="text-sm text-neutral-600">No rows yet. Add a user to get started.</p>
					{:else}
						<div class="w-full overflow-x-scroll">
							<table class="min-w-[70rem] border-collapse text-base">
								<thead>
									<tr class="border-b border-neutral-300 bg-neutral-100">
										<!-- Sticky, wider first column -->
										<th
											class="sticky left-0 z-20 w-72 min-w-[25rem] bg-neutral-100 px-4 py-3 text-left text-lg font-semibold"
										>
											Email
										</th>

										{#each adminColumns as col}
											<th class="min-w-[10rem] px-4 py-3 text-left text-lg font-semibold">
												{col.replace(/_/g, ' ')}
											</th>
										{/each}
									</tr>
								</thead>

								<tbody>
									{#each adminRows as row, i}
										<tr class="border-b border-neutral-200">
											<!-- Sticky first cell, extra bottom padding on last row -->
											<td
												class="sticky left-0 z-10 w-72 min-w-[20rem] bg-neutral-50 px-4 py-3"
												class:pb-10={i === adminRows.length - 1}
											>
												<input
													class="w-full rounded border border-neutral-300 px-2 py-1.5 text-base"
													bind:value={adminRows[i].email}
													placeholder="email@example.com"
												/>
											</td>

											{#each adminColumns as col}
												<td
													class="min-w-[10rem] px-4 py-3"
													class:pb-10={i === adminRows.length - 1}
												>
													<input
														class="w-full rounded border border-neutral-300 px-2 py-1.5 text-base"
														bind:value={adminRows[i][col]}
														placeholder={col}
													/>
												</td>
											{/each}
										</tr>
									{/each}
								</tbody>
							</table>
						</div>
					{/if}

					<div class="mt-3 flex flex-wrap gap-2 text-sm">
						<button
							type="button"
							class="rounded-lg border border-neutral-300 bg-white px-3 py-3  font-medium text-neutral-900 shadow-sm hover:bg-neutral-100"
							on:click={addAdminRow}
						>
							Add row
						</button>

						<button
							type="button"
							class="rounded-lg bg-sky-600 px-4 py-3  font-medium text-white shadow-sm hover:bg-sky-700 disabled:opacity-60"
							on:click={saveAdmin}
							disabled={adminLoading}
						>
							{adminLoading ? 'Saving…' : 'Save changes'}
						</button>
					</div>
				</section>
			{:else}
				<!-- MEMBER IDs VIEW -->
				<section class="mx-auto mt-6 grid max-w-3xl gap-4 md:grid-cols-2">
					{#each identifiers as item}
						<article
							class="flex flex-col gap-2 rounded-xl border border-neutral-200 bg-neutral-50 px-4 py-4"
						>
							<div>
								<p class="text-xs tracking-wide text-neutral-500 uppercase">
									{item.key}
								</p>
								<p class="font-mono text-xl break-all text-[#8D2417]">
									{item.value}
								</p>
							</div>

							<button
								type="button"
								class="mt-1 inline-flex items-center justify-center rounded-lg border border-neutral-300 bg-white px-3 py-2 text-sm font-medium text-neutral-900 shadow-sm transition hover:bg-neutral-100 disabled:opacity-70"
								on:click={() => copyIdentifier(item.key, item.value)}
							>
								{#if copiedKey === item.key}
									Added to clipboard
								{:else}
									Copy to clipboard
								{/if}
							</button>
						</article>
					{/each}
				</section>
			{/if}

			<div class="pt-4 text-center">
				<div class="flex flex-col justify-center gap-3 pt-4 sm:flex-row">
					<button
						type="button"
						class="inline-flex items-center justify-center rounded-xl border border-neutral-300 bg-white px-4 py-3 text-sm font-medium text-neutral-900 shadow-sm transition hover:bg-neutral-50"
						on:click={resetAll}
					>
						Look up another email
					</button>

					{#if isAdmin}
						<button
							type="button"
							class="inline-flex items-center justify-center rounded-xl px-8 py-3 text-sm font-medium text-white shadow-sm transition
        {buttonColorClasses}"
							on:click={toggleAdminView}
						>
							{showAdmin ? 'Back to Member IDs' : 'View Admin Table'}
						</button>
					{/if}
				</div>
			</div>
			<div class="pt-0 text-center">
				<button
					type="button"
					class="inline-flex items-center justify-center rounded-xl bg-[#8D2417] px-8 py-3 text-sm font-medium text-white shadow-sm transition hover:bg-neutral-900"
					on:click={handleSignOut}
				>
					Sign out
				</button>
			</div>
		{/if}
	</div>
</div>
