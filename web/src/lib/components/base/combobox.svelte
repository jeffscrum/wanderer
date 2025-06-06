<script module lang="ts">
    export type ComboboxItem = {
        text: string;
        value: any;
        icon?: string;
    };
</script>

<script lang="ts">
    import type { ChangeEventHandler } from "svelte/elements";
    import Chip from "./chip.svelte";

    interface Props {
        name?: string;
        icon?: string;
        label?: string;
        value?: string | ComboboxItem[];
        items?: ComboboxItem[];
        multiple?: boolean;
        chips?: boolean;
        placeholder?: string;
        extraClasses?: string;
        onchange?: ChangeEventHandler<HTMLInputElement>;
        onupdate?: (q: string) => void;
    }

    let {
        name = "",
        icon = "",
        label = "",
        multiple = false,
        value = $bindable(multiple ? [] : undefined),
        items = [],
        chips = false,
        placeholder = "",
        extraClasses = "",
        onchange,
        onupdate,
    }: Props = $props();

    let searching: boolean = $state(false);

    let inputValue: string = $state("");

    let dropDownOpen = $derived(
        ((multiple && inputValue.length > 0) ||
            (!multiple && (value as string)?.length > 0)) &&
            items.length > 0 &&
            searching,
    );

    $effect(() => {
        items;
        makeMatchesBold();
    });

    async function makeMatchesBold() {
        const dropdownMenu = document.querySelector(".menu");

        const relevantValue = multiple ? inputValue : (value as string);

        if (dropdownMenu) {
            for (let i = 0; i < dropdownMenu.children.length; i++) {
                const li = dropdownMenu.children[i];
                const textNode = li.getElementsByTagName("p")[0];
                if(!textNode) {
                    return
                }
                textNode.innerHTML = items[i].text;

                const text = textNode.innerText.replace(
                    new RegExp(relevantValue, "gi"),
                    (match) => `<strong>${match}</strong>`,
                );
                textNode.innerHTML = text;
            }
        }
    }

    async function onSearchType() {
        const relevantValue = multiple ? inputValue : (value as string);

        update(relevantValue);
    }

    function update(q: string) {
        onupdate?.(q);
    }

    function handleItemClick(e: Event, item: ComboboxItem) {
        e.stopPropagation();

        if (multiple) {
            if ((value as ComboboxItem[]).some((i) => i.text == item.text)) {
                return;
            }
            inputValue = "";
            value = [...(value as ComboboxItem[]), item];
        } else {
            value = item.value;
        }
    }

    function handleKeydown(e: KeyboardEvent) {
        if (!multiple) {
            return;
        }

        if (e.key == "Enter") {
            e.preventDefault();
            e.stopPropagation();
            if (!inputValue.length) {
                return;
            }
            if ((value as ComboboxItem[]).some((i) => i.text == inputValue)) {
                inputValue = "";
                return;
            }

            const matchingItemFromSuggestions = items.find(
                (i) => i.text == inputValue,
            );

            value = [
                ...(value as ComboboxItem[]),
                matchingItemFromSuggestions
                    ? matchingItemFromSuggestions
                    : {
                          text: inputValue,
                          value: null,
                      },
            ];

            inputValue = "";
        } else if (e.key == "Backspace" && inputValue.length == 0) {
            value = (value as ComboboxItem[]).filter(
                (_, i) => i !== value.length - 1,
            );
        }
    }

    function getInputValue() {
        if (multiple) {
            return inputValue;
        } else {
            return value as string;
        }
    }

    function setInputValue(v: string) {
        if (multiple) {
            inputValue = v;
        } else {
            value = v;
        }
    }
</script>

<div class="relative {extraClasses}">
    {#if label.length}
        <label for={name} class="text-sm font-medium pb-1">
            {label}
        </label>
    {/if}
    <div
        class="flex items-center gap-1 flex-wrap bg-input-background border border-input-border rounded-md p-3 transition-colors focus-within:border-input-border-focus focus-within:ring-0 w-full {extraClasses}"
    >
        {#if multiple}
            {#each value as ComboboxItem[] as v, i}
                {#if chips}
                    <Chip
                        text={v.text}
                        closable
                        onclick={(e) => {
                            value = (value as ComboboxItem[]).filter(
                                (_, idx) => i != idx,
                            );
                        }}
                    ></Chip>
                {:else}
                    <span
                        >{v.text}{i < (value as ComboboxItem[]).length - 1
                            ? ","
                            : ""}</span
                    >
                {/if}
            {/each}
        {/if}
        <input
            class="flex-1 min-w-24 bg-input-background focus:outline-none"
            type="search"
            {name}
            oninput={onSearchType}
            autocomplete="off"
            {onchange}
            placeholder={value.length ? undefined : placeholder}
            onfocusin={() => (searching = true)}
            onfocusout={() => (searching = false)}
            onkeydown={(e) => handleKeydown(e)}
            bind:value={getInputValue, setInputValue}
        />
    </div>

    {#if dropDownOpen}
        <ul
            class="menu absolute bg-menu-background border border-input-border rounded-xl shadow-md w-full max-h-64 overflow-x-hidden overflow-y-scroll"
            class:none={!dropDownOpen}
            style="z-index: 1001"
        >
            {#each items as item}
                <!-- svelte-ignore a11y_no_noninteractive_element_interactions -->
                <!-- svelte-ignore a11y_no_noninteractive_tabindex -->
                <li
                    class="menu-item flex items-center px-4 py-3 cursor-pointer hover:bg-menu-item-background-hover focus:bg-menu-item-background-focus transition-colors"
                    tabindex="0"
                    onmousedown={(e) => handleItemClick(e, item)}
                    onkeydown={(e) => handleItemClick(e, item)}
                >
                    {#if item.icon}
                        <i class="fa fa-{item.icon} mr-6"></i>
                    {/if}
                    <p class="text-ellipsis">{item.text}</p>
                </li>
            {/each}
        </ul>
    {/if}
</div>
