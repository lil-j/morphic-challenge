<script>
    import { fly } from 'svelte/transition';
    import { tick } from 'svelte';

    function animateHeightChange(node, { duration = 400 } = {}) {
        const oldHeight = node.offsetHeight;
        return {
            duration,
            css: t => {
                const newHeight = node.scrollHeight;
                const height = oldHeight + t * (newHeight - oldHeight);
                return `height: ${height}px`;
            }
        };
    }

    let allSettings = [
        {
            name: "Format",
            description: "Alter the aspect ratio of the image",
            color: "#ffd152",
            subsettings: [
                { name: "Square", description: "Generate a square image" },
                { name: "Portrait", description: "Generate a portrait image" },
                { name: "Landscape", description: "Generate a landscape image" },
            ]
        },
        {
            name: "Style",
            description: "Change the artistic style of the image",
            color: "#52a9ff",
            subsettings: [
                { name: "Anime", description: "Generate an anime-style image" },
                { name: "Portrait", description: "Generate a portrait-style image" },
                { name: "Landscape", description: "Generate a landscape-style image" },
                { name: "Abstract", description: "Generate an abstract-style image" },
                { name: "Realism", description: "Generate a realistic-style image" },
                { name: "Cartoon", description: "Generate a cartoon-style image" },
                { name: "Pixel", description: "Generate a pixel-style image" },
                { name: "Sketch", description: "Generate a sketch-style image" },
                { name: "Watercolor", description: "Generate a watercolor-style image" },
            ]
        },
        {
            name: "Model",
            description: "Choose between different stable diffusion models",
        },
        {
            name: "Model Type",
            description: "Choose what the model should prioritize while generating the image"
        },
    ];

    let settings = allSettings;
    let subsettings = [];
    let filteredSubsettings = [];

    let inputValue = '';
    let containsAtSign = false;
    let textAfterAtSign = '';
    let showAlternative = false;
    let showMain = true;
    let alternativeHeight = 0;
    let alternativeDiv;
    let editableDiv;

    $: {
        const lastAtIndex = inputValue.lastIndexOf('@');
        textAfterAtSign = lastAtIndex !== -1 ? inputValue.substring(lastAtIndex) : '';
        textAfterAtSign = textAfterAtSign.substring(1); // Remove the @ sign

        const lastChar = textAfterAtSign.charAt(textAfterAtSign.length - 1);
        containsAtSign = lastAtIndex !== -1 &&
            !textAfterAtSign.includes('\u00A0') &&
            !textAfterAtSign.includes(' ') &&
            lastChar !== ' ' &&
            lastChar !== '\u00A0';
    }

    $: (async () => {
        if (containsAtSign) {
            showAlternative = false;
            await tick();
            showMain = false;
            setTimeout(() => {
                if (containsAtSign) {
                    showAlternative = true;
                }
            }, 400);
        } else {
            showAlternative = false;
            setTimeout(() => {
                showAlternative = false;
                showMain = true;
            }, 400);
        }
    })();

    $: if (alternativeDiv) {
        alternativeHeight = alternativeDiv.offsetHeight;
    }

    $: if (showAlternative) {
        tick().then(() => {
            if (alternativeDiv) {
                alternativeHeight = alternativeDiv.offsetHeight;
            }
        });
    }

    $: {
        if (containsAtSign) {
            const colonIndex = textAfterAtSign.indexOf(':');
            if (colonIndex === -1) {
                // Filter main settings
                const baseSettings = allSettings.filter(setting => setting.name.toLowerCase().includes(textAfterAtSign.toLowerCase()));
                settings = baseSettings.map((setting, index) => ({...setting, hovered: index === 0}));
            } else {
                // Filter subsettings
                const mainSetting = textAfterAtSign.substring(0, colonIndex);
                const subSettingText = textAfterAtSign.substring(colonIndex + 1).trim();
                const setting = allSettings.find(setting => setting.name.toLowerCase() === mainSetting.toLowerCase());
                if (setting) {
                    subsettings = setting.subsettings;
                    filteredSubsettings = subsettings.filter(subsetting => subsetting.name.toLowerCase().includes(subSettingText.toLowerCase()));
                    settings = filteredSubsettings.map((subsetting, index) => ({...subsetting, hovered: index === 0}));
                }
            }
        } else {
            subsettings = [];
            settings = allSettings;
        }
    }

    function handleKeydown(event) {
        if (showAlternative) {
            const hoveredIndex = settings.findIndex(setting => setting.hovered);
            if (event.key === 'ArrowDown') {
                settings[hoveredIndex].hovered = false;
                settings[(hoveredIndex + 1) % settings.length].hovered = true;
            } else if (event.key === 'ArrowUp') {
                event.preventDefault();
                settings[hoveredIndex].hovered = false;
                settings[(hoveredIndex - 1 + settings.length) % settings.length].hovered = true;
            } else if (event.key === 'Enter') {
                event.preventDefault();  // Prevent the default action (new line)
                handleEnterKey();
            }
        }
    }

    async function handleEnterKey() {
        const selectedSetting = settings.find(setting => setting.hovered);
        const atIndex = inputValue.lastIndexOf('@');

        if (atIndex !== -1) {
            const colonIndex = inputValue.indexOf(':', atIndex);
            if (colonIndex !== -1) {
                // Append the subsetting correctly
                inputValue = inputValue.substring(0, colonIndex + 1) + selectedSetting.name + "\u00A0";
                // Hide the alternative div
                toggleAlternative();
            } else {
                // Replace the part after the last @ with the selected setting
                inputValue = inputValue.substring(0, atIndex + 1) + selectedSetting.name + ":";
            }
        } else {
            inputValue += selectedSetting.name + ": ";
        }

        if (selectedSetting.subsettings) {
            subsettings = selectedSetting.subsettings;
            settings = subsettings.map((subsetting, index) => ({...subsetting, hovered: index === 0}));
        } else {
            subsettings = [];
            settings = allSettings;
        }

        // Set alternativeHeight to a fixed value and ensure reactivity
        alternativeHeight = 384;
        await tick(); // Ensure DOM updates

        // Move cursor to the end of the contenteditable div
        setCursorToEnd(editableDiv);
        wrapSettingsInSpans(); // Ensure the span elements are correctly maintained
    }

    async function toggleAlternative() {
        if (showAlternative) {
            showAlternative = false;
            setTimeout(() => {
                showAlternative = false;
                showMain = true;
            }, 400);
        } else {
            showAlternative = false;
            await tick();
            showMain = false;
            setTimeout(() => {
                if (containsAtSign) {
                    showAlternative = true;
                }
            }, 400);
        }
    }

    function handleAlternativeMount(node) {
        alternativeDiv = node;
        alternativeHeight = node.offsetHeight;
        return {
            destroy() {
                alternativeHeight = 0;
                alternativeDiv = null;
            }
        }
    }

    function handleInput(event) {
        inputValue = event.target.innerText;
        wrapSettingsInSpans();
    }

    $: if (editableDiv && editableDiv.innerText !== inputValue) {
        editableDiv.innerText = inputValue;
        wrapSettingsInSpans();
    }

    function wrapSettingsInSpans() {
        const atSignPattern = /(@[^:]+:[^\s]*)(\s)/g;
        editableDiv.innerHTML = editableDiv.innerText.replace(atSignPattern, (match, p1, p2) => {
            const span = document.createElement('span');
            span.innerText = p1;
            // get setting name
            const settingName = p1.substring(1, p1.indexOf(':'));
            const setting = allSettings.find(setting => setting.name === settingName);
            if (setting) {
                span.style.color = setting.color;
                span.setAttribute('transition:wipe', '');
            }
            return span.outerHTML + p2;
        });
        setCursorToEnd(editableDiv);
    }

    function setCursorToEnd(el) {
        el.focus();
        const range = document.createRange();
        const sel = window.getSelection();
        range.selectNodeContents(el);
        range.collapse(false);
        sel.removeAllRanges();
        sel.addRange(range);
        el.onkeydown = backspaceKeyDown; // Add keydown event listener
    }

    function handleMouseover(event, index) {
        settings.forEach(setting => setting.hovered = false);
        settings[index].hovered = true;
    }

    function handleClick(event, index) {
        settings.forEach(setting => setting.hovered = false);
        settings[index].hovered = true;
        handleEnterKey();
    }
</script>

<svelte:window on:keydown={handleKeydown}/>

<div class="fixed bottom-12 w-full">
    <div class="max-w-2xl mx-auto w-full">
        <div class="relative">
            <div class="h-auto overflow-hidden">
                {#if showMain}
                    <div class="px-4" transition:fly={{ y: 105, duration: 300, opacity: 1 }}>
                        <div class="border border-custom-border-color border-b-0 w-full h-10 bg-neutral-900 rounded-2xl rounded-bl-none rounded-br-none">
                            <!-- Colors, Icons, Etc. -->
                        </div>
                    </div>
                    <div class="text-[12px] border-b-0 text-neutral-700 flex gap-4 border border-custom-border-color bg-neutral-900 rounded-2xl rounded-bl-none rounded-br-none w-full px-4 py-3"
                         transition:fly={{ y: 45, duration: 300, opacity: 1 }}>
                        <a class="text-neutral-100">Image</a>
                        <a>Video</a>
                        <a>Interpolation</a>
                    </div>
                {/if}
            </div>
            <div class="overflow-hidden" transition:animateHeightChange>
                {#if showAlternative}
                    <div use:handleAlternativeMount
                         class="p-2 border border-custom-border-color bg-neutral-900 rounded-2xl rounded-bl-none rounded-br-none w-full"
                         transition:fly={{ y: alternativeHeight, duration: 400 }}>
                        <div class="flex flex-col gap-2 p-2 max-h-96 overflow-y-auto">
                            {#each settings as setting, index}
                                <div class="flex text-xs justify-between {setting.hovered ? 'bg-neutral-800' : 'bg-neutral-800/0'} rounded-md p-2"
                                     on:mouseover={(event) => handleMouseover(event, index)}
                                     on:click={(event) => handleClick(event, index)}>
                                    <div>
                                        <p class="text-neutral-100">{setting.name}</p>
                                    </div>
                                    <div>
                                        <p class="text-neutral-400 text-[10px]">{setting.description}</p>
                                    </div>
                                </div>
                            {/each}
                        </div>
                    </div>
                {/if}
            </div>
            <div class="relative z-10 p-2 border border-custom-border-color bg-neutral-900 rounded-2xl rounded-tl-none rounded-tr-none w-full">
                <div class="grid grid-cols-7 gap-1.5">
                    <div class="col-span-6 border border-black rounded-[0.55rem]">
                        <div
                                contenteditable="true"
                                placeholder="Imagine.."
                                class="outline-none focus:ring-2 ring-neutral-200/20 transition duration-300 ease-in-out text-[14px] w-full border border-custom-border-color bg-neutral-900 text-neutral-100 rounded-lg p-2"
                                on:input={handleInput}
                                bind:this={editableDiv}
                        ></div>
                    </div>
                    <button class="bg-[#0075FF] text-white rounded-[0.55rem] text-[14px]">Generate</button>
                </div>
            </div>
        </div>
    </div>
</div>