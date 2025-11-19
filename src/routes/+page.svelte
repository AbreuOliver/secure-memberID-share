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

	// Restore session on page load: if user already logged in,
	// fetch their identifiers and go straight to success view.
	onMount(async () => {
		try {
			const { data, error } = await supabase.auth.getUser();
			if (error || !data.user) return;

			const { data: row, error: rowError } = await supabase
				.from('approved_user_info')
				.select('*')
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

			const { data: row, error: rowError } = await supabase
				.from('approved_user_info')
				.select('*')
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
			resetAll(); // your existing reset function
		}
	}
</script>

<div class="flex min-h-screen w-full items-center justify-center bg-white p-6 text-neutral-900">
	<div class="w-full max-w-3xl space-y-8">
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
				<h1 class="text-3xl font-semibold tracking-tight">Your Insurance Identifiers</h1>
				<p class="text-base text-neutral-700">
					These are the identifiers associated with your account. You can copy each one
					individually.
				</p>
			</header>

			{#if generalError}
				<div class="rounded-lg border border-red-200 bg-red-50 px-4 py-3 text-sm text-red-700">
					{generalError}
				</div>
			{/if}

			<section class="grid gap-4 md:grid-cols-2">
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

			<div class="pt-4 text-center">
				<div class="flex flex-col justify-center gap-3 pt-4 sm:flex-row">
					<button
						type="button"
						class="inline-flex items-center justify-center rounded-xl border border-neutral-300 bg-white px-4 py-3 text-sm font-medium text-neutral-900 shadow-sm transition hover:bg-neutral-50"
						on:click={resetAll}
					>
						Look up another email
					</button>

					<button
						type="button"
						class="inline-flex items-center justify-center rounded-xl bg-[#8D2417] px-8 py-3 text-sm font-medium text-white shadow-sm transition hover:bg-neutral-900"
						on:click={handleSignOut}
					>
						Sign out
					</button>
				</div>
			</div>
		{/if}
	</div>
</div>
