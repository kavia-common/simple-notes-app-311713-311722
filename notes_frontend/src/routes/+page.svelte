<script lang="ts">
	type Note = {
		id: string;
		title: string;
		content: string;
		createdAt: string;
		updatedAt: string;
	};

	// Backend base URL.
	// IMPORTANT: `localhost` only works for local dev; in hosted/preview environments it breaks.
	// Configure `PUBLIC_API_BASE_URL` at build/runtime to point to the deployed backend (e.g. https://<host>:3001).
	const API_BASE =
		(import.meta as any).env?.PUBLIC_API_BASE_URL?.toString()?.replace(/\/$/, "") ?? "http://localhost:3001";

	let notes: Note[] = [];
	let loading = true;
	let saving = false;
	let deletingId: string | null = null;

	let selectedId: string | null = null;

	// Form state (used for both create & edit)
	let formTitle = "";
	let formContent = "";

	let errorMessage: string | null = null;

	function formatDate(iso: string) {
		try {
			return new Date(iso).toLocaleString();
		} catch {
			return iso;
		}
	}

	async function api<T>(path: string, init?: RequestInit): Promise<T> {
		const res = await fetch(`${API_BASE}${path}`, {
			headers: { "Content-Type": "application/json", ...(init?.headers ?? {}) },
			...init
		});

		if (!res.ok) {
			const text = await res.text().catch(() => "");
			throw new Error(text || `Request failed: ${res.status}`);
		}

		// 204 No Content
		if (res.status === 204) return undefined as T;

		return (await res.json()) as T;
	}

	async function loadNotes() {
		loading = true;
		errorMessage = null;

		try {
			notes = await api<Note[]>("/notes");
		} catch (e) {
			errorMessage = e instanceof Error ? e.message : "Failed to load notes";
		} finally {
			loading = false;
		}
	}

	function startCreate() {
		selectedId = null;
		formTitle = "";
		formContent = "";
		errorMessage = null;
	}

	function startEdit(note: Note) {
		selectedId = note.id;
		formTitle = note.title;
		formContent = note.content;
		errorMessage = null;
	}

	async function save() {
		errorMessage = null;

		const title = formTitle.trim();
		const content = formContent ?? "";

		if (!title) {
			errorMessage = "Title is required.";
			return;
		}

		saving = true;
		try {
			if (selectedId) {
				const updated = await api<Note>(`/notes/${selectedId}`, {
					method: "PUT",
					body: JSON.stringify({ title, content })
				});
				notes = notes.map((n) => (n.id === updated.id ? updated : n));
			} else {
				const created = await api<Note>("/notes", {
					method: "POST",
					body: JSON.stringify({ title, content })
				});
				notes = [created, ...notes];
				selectedId = created.id;
			}

			// Keep list sorted by updatedAt desc
			notes = [...notes].sort(
				(a, b) => new Date(b.updatedAt).getTime() - new Date(a.updatedAt).getTime()
			);
		} catch (e) {
			errorMessage = e instanceof Error ? e.message : "Failed to save note";
		} finally {
			saving = false;
		}
	}

	async function deleteNote(id: string) {
		errorMessage = null;
		deletingId = id;
		try {
			await api<void>(`/notes/${id}`, { method: "DELETE" });
			notes = notes.filter((n) => n.id !== id);
			if (selectedId === id) startCreate();
		} catch (e) {
			errorMessage = e instanceof Error ? e.message : "Failed to delete note";
		} finally {
			deletingId = null;
		}
	}

	$effect(() => {
		loadNotes();
	});
</script>

<svelte:head>
	<title>Notes</title>
</svelte:head>

<header class="topbar">
	<div class="container topbar-inner">
		<div class="brand">
			<div class="logo" aria-hidden="true">N</div>
			<div>
				<div class="brand-title">Notes</div>
				<div class="brand-subtitle muted">Simple CRUD notes app</div>
			</div>
		</div>

		<div class="actions">
			<button class="btn btn-primary" on:click={startCreate}>New note</button>
			<a class="btn btn-ghost" href={`${API_BASE}/docs`} target="_blank" rel="noreferrer">
				API docs
			</a>
		</div>
	</div>
</header>

<div class="container page">
	{#if errorMessage}
		<div class="alert" role="alert">
			<strong>Error:</strong> {errorMessage}
		</div>
	{/if}

	<div class="grid">
		<section class="card list" aria-label="Notes list">
			<div class="panel-header">
				<h2>All notes</h2>
				<span class="muted">{notes.length}</span>
			</div>
			<div class="divider" />

			{#if loading}
				<div class="empty muted">Loading…</div>
			{:else if notes.length === 0}
				<div class="empty">
					<div class="empty-title">No notes yet</div>
					<div class="muted">Create your first note to get started.</div>
					<div style="height: 12px" />
					<button class="btn btn-primary" on:click={startCreate}>Create note</button>
				</div>
			{:else}
				<ul class="notes">
					{#each notes as n (n.id)}
						<li>
							<button
								class="note-row {selectedId === n.id ? 'active' : ''}"
								on:click={() => startEdit(n)}
							>
								<div class="note-title">{n.title}</div>
								<div class="note-meta muted">
									Updated {formatDate(n.updatedAt)}
								</div>
							</button>
						</li>
					{/each}
				</ul>
			{/if}
		</section>

		<section class="card editor" aria-label="Note editor">
			<div class="panel-header">
				<h2>{selectedId ? "Edit note" : "Create note"}</h2>
				<div class="editor-actions">
					{#if selectedId}
						<button
							class="btn btn-danger"
							disabled={deletingId === selectedId}
							on:click={() => deleteNote(selectedId!)}
						>
							{deletingId === selectedId ? "Deleting…" : "Delete"}
						</button>
					{/if}
					<button class="btn btn-success" disabled={saving} on:click={save}>
						{saving ? "Saving…" : "Save"}
					</button>
				</div>
			</div>
			<div class="divider" />

			<div class="form">
				<label class="field">
					<div class="label">Title</div>
					<input class="input" placeholder="Untitled note" bind:value={formTitle} />
				</label>

				<label class="field">
					<div class="label">Content</div>
					<textarea class="textarea" placeholder="Write something…" bind:value={formContent} />
				</label>

				{#if selectedId}
					<div class="meta muted">
						<div>Created: {notes.find((n) => n.id === selectedId)?.createdAt
							? formatDate(notes.find((n) => n.id === selectedId)!.createdAt)
							: "—"}</div>
						<div>Updated: {notes.find((n) => n.id === selectedId)?.updatedAt
							? formatDate(notes.find((n) => n.id === selectedId)!.updatedAt)
							: "—"}</div>
					</div>
				{/if}
			</div>
		</section>
	</div>
</div>

<style>
	.topbar {
		position: sticky;
		top: 0;
		z-index: 5;
		background: rgba(249, 250, 251, 0.85);
		backdrop-filter: blur(10px);
		border-bottom: 1px solid var(--color-border);
	}

	.topbar-inner {
		display: flex;
		align-items: center;
		justify-content: space-between;
		padding: 14px 0;
		gap: 16px;
	}

	.brand {
		display: flex;
		align-items: center;
		gap: 12px;
		min-width: 220px;
	}

	.logo {
		width: 36px;
		height: 36px;
		border-radius: 10px;
		display: grid;
		place-items: center;
		font-weight: 800;
		color: #fff;
		background: linear-gradient(135deg, var(--color-primary), var(--color-success));
	}

	.brand-title {
		font-weight: 700;
		line-height: 1.1;
	}

	.brand-subtitle {
		font-size: 12px;
	}

	.actions {
		display: flex;
		gap: 10px;
		flex-wrap: wrap;
		justify-content: flex-end;
	}

	.page {
		padding: 20px 0 28px;
	}

	.alert {
		background: rgba(239, 68, 68, 0.08);
		border: 1px solid rgba(239, 68, 68, 0.3);
		color: var(--color-text);
		padding: 12px 14px;
		border-radius: 12px;
		margin-bottom: 16px;
	}

	.grid {
		display: grid;
		grid-template-columns: 360px 1fr;
		gap: 16px;
		align-items: start;
	}

	@media (max-width: 920px) {
		.grid {
			grid-template-columns: 1fr;
		}
	}

	.panel-header {
		padding: 14px 14px;
		display: flex;
		align-items: center;
		justify-content: space-between;
		gap: 12px;
	}

	.panel-header h2 {
		margin: 0;
		font-size: 14px;
		text-transform: uppercase;
		letter-spacing: 0.08em;
		color: rgba(17, 24, 39, 0.75);
	}

	.list {
		overflow: hidden;
	}

	.notes {
		list-style: none;
		padding: 8px;
		margin: 0;
		display: flex;
		flex-direction: column;
		gap: 8px;
	}

	.note-row {
		width: 100%;
		text-align: left;
		border: 1px solid var(--color-border);
		background: #fff;
		border-radius: 12px;
		padding: 12px 12px;
		cursor: pointer;
		transition: background-color 0.12s ease, border-color 0.12s ease;
	}

	.note-row:hover {
		background: rgba(59, 130, 246, 0.04);
		border-color: rgba(59, 130, 246, 0.35);
	}

	.note-row.active {
		background: rgba(59, 130, 246, 0.08);
		border-color: rgba(59, 130, 246, 0.55);
	}

	.note-title {
		font-weight: 650;
		margin-bottom: 4px;
	}

	.note-meta {
		font-size: 12px;
	}

	.empty {
		padding: 18px 14px;
	}

	.empty-title {
		font-weight: 700;
		margin-bottom: 6px;
	}

	.editor-actions {
		display: flex;
		gap: 10px;
	}

	.form {
		padding: 14px;
		display: flex;
		flex-direction: column;
		gap: 12px;
	}

	.field .label {
		font-size: 12px;
		font-weight: 650;
		margin-bottom: 6px;
		color: rgba(17, 24, 39, 0.75);
	}

	.meta {
		display: flex;
		gap: 16px;
		flex-wrap: wrap;
		font-size: 12px;
	}
</style>
