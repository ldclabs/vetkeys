<script lang="ts">
    import { onDestroy } from "svelte";
    import { Editor, placeholder } from "typewriter-editor";
    import { auth } from "../store/auth";
    import { draft } from "../store/draft";
    import { setPassword, refreshVaults } from "../store/vaults";
    import { addNotification, showError } from "../store/notifications";
    import Header from "./Header.svelte";
    import PasswordEditor from "./PasswordEditor.svelte";
    import { Principal } from "@dfinity/principal";

    let creating = false;
    let vaultOwner =
        $auth.state === "initialized"
            ? $auth.client.getIdentity().getPrincipal().toText()
            : Principal.anonymous().toText();
    let vaultName = "";
    let passwordName = "";
    let url: string = "";

    let tagsInput: string = "";
    let tags: string[] = [];
    // Convert between string and array when the input changes
    export function handleTagsInput() {
        // Split the input string by commas, trim whitespace, and filter empty strings
        tags = [
            ...new Set(
                tagsInput
                    .split(",")
                    .map((tag) => tag.trim())
                    .filter((tag) => tag !== ""),
            ),
        ];
    }

    const editor = new Editor({
        modules: {
            placeholder: placeholder("Enter password..."),
        },
        html: $draft.content,
    });

    async function add() {
        if ($auth.state !== "initialized") {
            return;
        }

        creating = true;

        await setPassword(
            Principal.fromText(vaultOwner),
            vaultName,
            passwordName,
            editor.getText(),
            url,
            tags,
            $auth.passwordManager,
        )
            .catch((e: Error) => {
                showError(e, "Could not add password.");
            })
            .finally(() => {
                creating = false;
            });

        // if creation was successful, reset the editor
        editor.setHTML("");

        addNotification({
            type: "success",
            message: "Password added successfully",
        });

        // refresh passwords in the background
        refreshVaults(
            $auth.client.getIdentity().getPrincipal(),
            $auth.passwordManager,
        ).catch((e: Error) => showError(e, "Could not refresh passwords."));
    }

    function saveDraft() {
        draft.set({
            content: editor.getText(),
        });
    }

    onDestroy(saveDraft);
</script>

<svelte:window on:beforeunload={saveDraft} />

<Header>
    <span slot="title"> New password </span>
</Header>

<main class="p-4">
    <!-- Add these input fields -->
    <div class="mb-3">
        <input
            type="text"
            bind:value={vaultOwner}
            placeholder="Enter vault owner"
            class="input input-bordered mb-3 w-full"
        />
        <input
            type="text"
            bind:value={vaultName}
            placeholder="Enter vault name"
            class="input input-bordered w-full"
        />
        <input
            type="text"
            bind:value={passwordName}
            placeholder="Enter password name"
            class="input input-bordered w-full"
        />
        <input
            type="text"
            bind:value={url}
            placeholder="Enter optional URL"
            class="input input-bordered w-full"
        />
        <input
            type="text"
            bind:value={tagsInput}
            on:input={handleTagsInput}
            placeholder="Enter optional tags"
            class="input input-bordered w-full"
        />
    </div>
    <PasswordEditor {editor} class="mb-3" disabled={creating} />
    <button
        class="btn btn-primary mt-6 {creating ? 'loading' : ''}"
        disabled={creating}
        on:click={add}>{creating ? "Adding..." : "Add password"}</button
    >
</main>
