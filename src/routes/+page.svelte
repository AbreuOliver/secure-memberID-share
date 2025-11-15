<script lang="ts">
  import { onMount } from "svelte";
  import { supabase } from "$lib/supabase"; // adjust path/name if needed

  type Step = "idle" | "email" | "code" | "success";

  let modalOpen = false;
  let step: Step = "idle";

  let email = "";
  let code = "";
  let identifier: string | null = null;

  let sending = false;
  let verifying = false;
  let fetchingId = false;

  let emailError: string | null = null;
  let codeError: string | null = null;
  let generalError: string | null = null;

  let emailInputEl: HTMLInputElement | null = null;
  let codeInputEl: HTMLInputElement | null = null;

  onMount(() => {
    // e.g., check existing session if you want
  });

  function openModal() {
    modalOpen = true;
    generalError = null;
    emailError = null;
    codeError = null;
    step = "email";
    email = "";
    code = "";
    identifier = null;
  }

  function closeModal() {
    if (sending || verifying || fetchingId) return;
    modalOpen = false;
    step = "idle";
  }

  function validateEmail(value: string): boolean {
    const trimmed = value.trim();
    if (!trimmed) {
      emailError = "Please enter your email address.";
      return false;
    }
    const pattern = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    if (!pattern.test(trimmed)) {
      emailError = "Please enter a valid email address.";
      return false;
    }
    emailError = null;
    return true;
  }

  function validateCode(value: string): boolean {
    const trimmed = value.trim();
    if (!trimmed) {
      codeError = "Please enter the 6-digit code.";
      return false;
    }
    if (!/^\d{6}$/.test(trimmed)) {
      codeError = "The code must be exactly 6 digits.";
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
      const { data, error } = await supabase.functions.invoke("request-otp", {
        body: { email: email.trim() }
      });

      if (error) {
        console.error(error);
        generalError = error.message ?? "Unable to send code. Please try again.";
        return;
      }

      step = "code";

      setTimeout(() => {
        if (codeInputEl) codeInputEl.focus();
      }, 0);
    } catch (err: unknown) {
      console.error(err);
      generalError = "Something went wrong while sending the code.";
    } finally {
      sending = false;
    }
  }

  async function handleCodeSubmit() {
    generalError = null;
    if (!validateCode(code)) return;

    verifying = true;
    fetchingId = true;

    try {
      // 1) Verify the OTP via your Edge Function
      const { data, error } = await supabase.functions.invoke(
        "verify-otp",
        {
          body: {
            email: email.trim(),
            code: code.trim()
          }
        }
      );

      if (error) {
        console.error(error);
        generalError = error.message ?? "Invalid or expired code. Please try again.";
        return;
      }

      // 2) After OTP is valid, fetch this user's row from approved_user_info
      const { data: row, error: rowError } = await supabase
        .from("approved_user_info")
        .select("*")
        .single(); // RLS ensures this is only *their* row

      if (rowError) {
        console.error(rowError);
        generalError = "Unable to load your insurance info.";
        return;
      }

      // Adjust column name if needed (e.g. aflac_dental, etc.)
      identifier = row.insurance_number ?? null;

      if (!identifier) {
        generalError = "We were unable to retrieve your identifier. Please contact support.";
        return;
      }

      step = "success";
    } catch (err: unknown) {
      console.error(err);
      generalError = "Something went wrong while verifying your code.";
    } finally {
      verifying = false;
      fetchingId = false;
    }
  }

  function handleKeydown(event: KeyboardEvent) {
    if (!modalOpen) return;
    if (event.key === "Escape") {
      event.stopPropagation();
      closeModal();
    }
  }

  function copyIdentifier() {
    if (!identifier) return;
    navigator.clipboard
      .writeText(identifier)
      .catch((err) => console.error("Failed to copy identifier:", err));
  }
</script>


<svelte:window on:keydown={handleKeydown} />

<!-- Outer wrapper for the widget (this is what would show inside the iframe) -->
<div class="min-h-screen w-full text-neutral-950 bg-white flex items-center justify-center p-4">
  <div class="w-full max-w-lg space-y-4 text-center">
    <h1 class="text-3xl font-semibold tracking-tight">
      Access Your Insurance Info
    </h1>
    <p class="text-lg my-6 text-neutral-00">
      Enter your RCA/BBC email address to receive a one-time code in order to securely
      view your insurance info.
    </p>

    <button
      type="button"
      class="inline-flex items-center justify-center rounded-xl bg-[#8D2417] px-6 py-4 text-xl font-medium text-white shadow-sm transition hover:bg-sky-400 focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-sky-400 focus-visible:ring-offset-2 focus-visible:ring-offset-neutral-950 capitalize"
      on:click={openModal}
    >
      Get started
    </button>
  </div>

  {#if modalOpen}
    <!-- Overlay -->
    <!-- svelte-ignore a11y_click_events_have_key_events -->
    <!-- svelte-ignore a11y_no_static_element_interactions -->
    <div
      class="fixed inset-0 z-40 bg-black/60 backdrop-blur-sm"
      on:click={() => closeModal()}
    ></div>

    <!-- Dialog -->
    <div
      class="fixed inset-0 z-50 flex items-center justify-center p-4"
      aria-modal="true"
      role="dialog"
    >
      <!-- svelte-ignore a11y_click_events_have_key_events -->
      <!-- svelte-ignore a11y_no_static_element_interactions -->
      <div
        class="relative w-full max-w-md rounded-xl bg-neutral-900 border border-neutral-800 shadow-xl p-5 sm:p-6"
        on:click|stopPropagation
      >
        <!-- Close button -->
        <button
          type="button"
          class="absolute right-3 top-3 rounded-full p-1 text-neutral-500 hover:text-neutral-200 focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-sky-400"
          on:click={closeModal}
          aria-label="Close"
        >
          <span class="sr-only">Close</span>
          <svg
            class="h-4 w-4"
            viewBox="0 0 24 24"
            fill="none"
            stroke="currentColor"
            stroke-width="2"
            stroke-linecap="round"
            stroke-linejoin="round"
            aria-hidden="true"
          >
            <line x1="18" y1="6" x2="6" y2="18" />
            <line x1="6" y1="6" x2="18" y2="18" />
          </svg>
        </button>

        <!-- Header -->
        <div class="space-y-1 mb-4">
          {#if step === "email" || step === "idle"}
            <h2 class="text-lg font-semibold">Enter your email</h2>
            <p class="text-xs text-neutral-400">
              We’ll check if your email is approved and send a one-time code.
            </p>
          {:else if step === "code"}
            <h2 class="text-lg font-semibold">Enter the 6-digit code</h2>
            <p class="text-xs text-neutral-400">
              Check your email for a code. Enter it here to verify your identity.
            </p>
          {:else if step === "success"}
            <h2 class="text-lg font-semibold">Your identifier</h2>
            <p class="text-xs text-neutral-400">
              Keep this identifier private. You can copy it for your records.
            </p>
          {/if}
        </div>

        {#if generalError}
          <div class="mb-3 rounded-md border border-red-400/50 bg-red-500/10 px-3 py-2 text-xs text-red-200">
            {generalError}
          </div>
        {/if}

        <!-- Email step -->
        {#if step === "email" || step === "idle"}
          <form
            class="space-y-3"
            on:submit|preventDefault={handleEmailSubmit}
          >
            <div class="space-y-1">
              <label for="email" class="block text-xs font-medium text-neutral-200">
                Email address
              </label>
              <input
                id="email"
                type="email"
                class="block w-full rounded-md border border-neutral-700 bg-neutral-950 px-3 py-2 text-sm text-neutral-50 placeholder:text-neutral-500 shadow-sm focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-sky-400 focus-visible:ring-offset-2 focus-visible:ring-offset-neutral-900"
                bind:value={email}
                bind:this={emailInputEl}
                autocomplete="email"
                spellcheck="false"
              />
              {#if emailError}
                <p class="text-[11px] text-red-300 mt-1">{emailError}</p>
              {/if}
            </div>

            <button
              type="submit"
              class="inline-flex w-full items-center justify-center rounded-md bg-sky-500 px-3 py-2 text-sm font-medium text-white shadow-sm transition hover:bg-sky-400 disabled:cursor-not-allowed disabled:opacity-60 focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-sky-400 focus-visible:ring-offset-2 focus-visible:ring-offset-neutral-900"
              disabled={sending}
            >
              {#if sending}
                <span class="mr-2 inline-block h-4 w-4 animate-spin rounded-full border-2 border-white/40 border-t-transparent"></span>
                Sending code…
              {:else}
                Send code
              {/if}
            </button>
          </form>
        {/if}

        <!-- Code step -->
        {#if step === "code"}
          <form
            class="space-y-3"
            on:submit|preventDefault={handleCodeSubmit}
          >
            <div class="space-y-1">
              <label for="code" class="block text-xs font-medium text-neutral-200">
                6-digit code
              </label>
              <input
                id="code"
                type="text"
                inputmode="numeric"
                maxlength="6"
                class="block w-full rounded-md border border-neutral-700 bg-neutral-950 px-3 py-2 text-sm text-neutral-50 text-center tracking-[0.3em] tabular-nums placeholder:text-neutral-500 shadow-sm focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-sky-400 focus-visible:ring-offset-2 focus-visible:ring-offset-neutral-900"
                bind:value={code}
                bind:this={codeInputEl}
                autocomplete="one-time-code"
              />
              {#if codeError}
                <p class="text-[11px] text-red-300 mt-1">{codeError}</p>
              {/if}
            </div>

            <button
              type="submit"
              class="inline-flex w-full items-center justify-center rounded-md bg-sky-500 px-3 py-2 text-sm font-medium text-white shadow-sm transition hover:bg-sky-400 disabled:cursor-not-allowed disabled:opacity-60 focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-sky-400 focus-visible:ring-offset-2 focus-visible:ring-offset-neutral-900"
              disabled={verifying}
            >
              {#if verifying}
                <span class="mr-2 inline-block h-4 w-4 animate-spin rounded-full border-2 border-white/40 border-t-transparent"></span>
                Verifying…
              {:else}
                Verify code
              {/if}
            </button>

            <button
              type="button"
              class="w-full text-center text-[11px] text-neutral-400 underline underline-offset-2 hover:text-neutral-200"
              on:click={() => {
                // Allow user to re-enter email / request new code
                step = "email";
                code = "";
                codeError = null;
              }}
            >
              Use a different email or request a new code
            </button>
          </form>
        {/if}

        <!-- Success step -->
        {#if step === "success" && identifier}
          <div class="space-y-4">
            <div class="rounded-lg border border-neutral-700 bg-neutral-950 px-3 py-3">
              <p class="text-[11px] uppercase tracking-wide text-neutral-500 mb-1">
                Your identifier
              </p>
              <p class="font-mono text-base text-sky-300 select-all break-all">
                {identifier}
              </p>
            </div>

            <div class="flex flex-col gap-2 sm:flex-row">
              <button
                type="button"
                class="inline-flex flex-1 items-center justify-center rounded-md bg-sky-500 px-3 py-2 text-sm font-medium text-white shadow-sm transition hover:bg-sky-400 focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-sky-400 focus-visible:ring-offset-2 focus-visible:ring-offset-neutral-900"
                on:click={copyIdentifier}
              >
                Copy identifier
              </button>

              <button
                type="button"
                class="inline-flex flex-1 items-center justify-center rounded-md border border-neutral-700 bg-neutral-950 px-3 py-2 text-sm font-medium text-neutral-100 shadow-sm transition hover:bg-neutral-800 focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-sky-400 focus-visible:ring-offset-2 focus-visible:ring-offset-neutral-900"
                on:click={closeModal}
              >
                Close
              </button>
            </div>
          </div>
        {/if}
      </div>
    </div>
  {/if}
</div>
