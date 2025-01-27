<!--
@component

### Editor

`textarea` element with controls to add content and keyboard shortcuts. Compared to other WYSIWYG editors, the `valueTextarea` is just a `string`, so you can easily store it in a database or manipulate it without learning a separate API. 

- This component is used to create [Typo](https://typo.robino.dev)

@props

- `classButton` - `class` of all the `button` elements
- `classControls` - `class` of the `div` that wraps the controls
- `classTextarea` - `class` of the `textarea` element
- `contentElements` - an array of `EditorContentElement`s for the controls
- `idControls` - `id` of the `div` that wraps the controls
- `idTextarea` - `id` of the `textarea` element
- `keyPairs` - keys that will auto-close if typed, value is their closing character
- `nameTextarea` - `name` of the `textarea` element
- `placeholderTextarea` - `placeholder` of the `textarea` element
- `selectionStartTextarea` - `selectionStart` value of the `textarea`
- `valueTextarea` - `value` of the `textarea` element

@Events

| name    | event                   |
| ------- | ----------------------- |
| `input` | forward from `textarea` |

@example

```svelte
<script lang="ts">
	import { Editor } from "drab";
</script>

<Editor
	classButton="button button-primary"
	classControls="flex gap-2"
	classTextarea="input w-full h-36 mb-2"
	placeholderTextarea="asterisk: ctrl+i, anchor: ctrl+["
	contentElements={[
		{
			title: "Bullet",
			text: "- ",
			display: "block",
			icon: "Bullet",
		},
		{
			title: "Italic",
			text: "*",
			display: "wrap",
			icon: "Italic",
			key: "i",
			class: "italic",
		},
		{
			title: "Anchor",
			text: "[text](href)",
			display: "inline",
			icon: "Anchor",
			key: "[",
		},
	]}
/>
```
-->

<script lang="ts">
	import { onMount, type ComponentType } from "svelte";

	/**
	 * - `EditorContentElement` to pass into the `contentElements` array prop
	 * - `contentElements` prop creates a list of button controls to insert
	 * text into the `textarea`
	 */
	interface EditorContentElement {
		/** title of element and it's corresponding button */
		title: string;

		/** text to add */
		text: string;

		/** controls how the text is added */
		display: "inline" | "block" | "wrap";

		/** contents of the `button` */
		icon: string | ComponentType;

		/** keyboard shortcut */
		key?: string;

		/** class to apply to the specific `button` */
		class?: string;
	}

	/** an array of `EditorContentElement`s for the controls */
	export let contentElements: EditorContentElement[] = [];

	/** `value` of the `textarea` element */
	export let valueTextarea = "";

	/** `placeholder` of the `textarea` element */
	export let placeholderTextarea = "";

	/** `class` of the `textarea` element */
	export let classTextarea = "";

	/** `id` of the `textarea` element */
	export let idTextarea = "";

	/** `name` of the `textarea` element */
	export let nameTextarea = "";

	/** `class` of all the `button` elements */
	export let classButton = "";

	/** `class` of the `div` that wraps the controls */
	export let classControls = "";

	/** `id` of the `div` that wraps the controls */
	export let idControls = "";

	/** `selectionStart` value of the `textarea` */
	export let selectionStartTextarea = 0;

	/** keys that will auto-close if typed, value is their closing character */
	export let keyPairs: { [key: string]: string } = {
		"(": ")",
		"{": "}",
		"[": "]",
		"<": ">",
		'"': '"',
		"`": "`",
	};

	/** set to `true` on the client */
	let clientJs = false;

	let textArea: HTMLTextAreaElement;

	// add any `display: "wrap"` text from `contentElements` to `keyPairs`
	contentElements.forEach((el) => {
		if (el.display === "wrap") keyPairs[el.text] = el.text;
	});

	/** array of keyPair characters that have been opened */
	let openChars: string[] = [];

	/**
	 * - insert character into string at index
	 *
	 * @param str string to insert into
	 * @param char characters to insert into `str`
	 * @param index where to insert the characters
	 * @returns the new string
	 */
	const insertChar = (str: string, char: string, index: number) => {
		return str.slice(0, index) + char + str.slice(index);
	};

	/**
	 * - remove char from string at index
	 *
	 * @param str string to remove the character from
	 * @param index index of character to remove
	 * @returns the new string
	 */
	const removeChar = (str: string, index: number) => {
		return str.slice(0, index) + str.slice(index + 1);
	};

	/**
	 * - Inserts text into the `textarea` based on the `display` property of
	 * the `ContentElement`.
	 *
	 * @param el the content element
	 * @param selectionStart current start position the selection
	 * @param selectionEnd current end position of the selection
	 */
	const insertText = async (
		el: EditorContentElement,
		selectionStart: number,
		selectionEnd: number,
	) => {
		if (el.display === "inline") {
			// insert at current position
			valueTextarea = `${valueTextarea.slice(0, selectionEnd)}${
				el.text
			}${valueTextarea.slice(selectionEnd)}`;
		} else if (el.display === "wrap") {
			valueTextarea = insertChar(valueTextarea, el.text, selectionStart);
			valueTextarea = insertChar(
				valueTextarea,
				keyPairs[el.text],
				selectionEnd + el.text.length,
			);
			// if single char, add to opened
			if (el.text.length < 2) openChars.push(el.text);
		} else if (el.display === "block") {
			const { lines, lineNumber } = getLineInfo();
			// add the string to the beginning of the line
			if (lines[lineNumber].startsWith(el.text[0])) {
				// avoids `# # # `, instead adds trimmed => `### `
				lines[lineNumber] = el.text.trim() + lines[lineNumber];
			} else {
				lines[lineNumber] = el.text + lines[lineNumber];
			}
			valueTextarea = lines.join("\n");
		}
	};

	/**
	 * - Sets the caret position after text is inserted based on
	 * the length of the text.
	 * - Highlights text if the content contains any letters.
	 *
	 * @param text
	 * @param selectionStart current start position the selection
	 * @param selectionEnd current end position of the selection
	 */
	const setCaretPosition = async (
		text: string,
		selectionStart: number,
		selectionEnd: number,
	) => {
		let startPos = 0;
		let endPos = 0;
		if (/[a-z]/i.test(text)) {
			// if string contains letters, highlight the first word
			for (let i = selectionEnd; i < valueTextarea.length; i++) {
				if (valueTextarea[i].match(/[a-z]/i)) {
					if (!startPos) {
						startPos = i;
					} else {
						endPos = i + 1;
					}
				} else if (startPos) {
					break;
				}
			}
		} else {
			// leave the cursor in place
			startPos = selectionStart + text.length;
			endPos = selectionEnd + text.length;
		}

		textArea.setSelectionRange(startPos, endPos);
		textArea.focus();
	};

	/**
	 * - Inserts the text and then sets the caret position
	 * based on the `ContentElement` selected.
	 *
	 * @param el selected content element
	 */
	const addContent = async (el: EditorContentElement) => {
		const selectionEnd = textArea.selectionEnd;
		const selectionStart = textArea.selectionStart;

		await insertText(el, selectionStart, selectionEnd);
		setCaretPosition(el.text, selectionStart, selectionEnd);
	};

	/**
	 * - Runs the keyboard shortcut for any element that matches the event
	 * - Closes opening characters, types over instead of inserting
	 *
	 * @param keyboardEvent KeyboardEvent
	 */
	const onKeyDown = async (e: KeyboardEvent) => {
		const resetKeys = ["ArrowUp", "ArrowDown", "Delete"];
		const nextChar = valueTextarea[textArea.selectionEnd];
		if (resetKeys.includes(e.key)) {
			// reset
			openChars = [];
		} else if (e.key === "Backspace") {
			const prevChar = valueTextarea[textArea.selectionStart - 1];
			if (prevChar in keyPairs && nextChar === keyPairs[prevChar]) {
				e.preventDefault();
				const start = textArea.selectionStart - 1;
				const end = textArea.selectionEnd - 1;
				valueTextarea = removeChar(valueTextarea, start);
				valueTextarea = removeChar(valueTextarea, end);
				setTimeout(() => {
					textArea.setSelectionRange(start, end);
				}, 0);
				openChars.pop();
			}
			if (
				prevChar === "\n" &&
				textArea.selectionStart === textArea.selectionEnd
			) {
				e.preventDefault();
				const newPos = textArea.selectionStart - 1;
				const { lineNumber } = getLineInfo();
				correctFollowing(lineNumber, true);
				valueTextarea = removeChar(valueTextarea, newPos);
				setTimeout(async () => {
					textArea.setSelectionRange(newPos, newPos);
				}, 0);
			}
		} else if (e.key === "Tab") {
			if (getCurrentBlock() % 2 !== 0) {
				// if caret is inside of a codeblock, indent
				e.preventDefault();
				await addContent({
					display: "inline",
					text: "\t",
					icon: "tab",
					title: "tab",
				});
			}
		} else if (e.key === "Enter") {
			// autocomplete start of next line if block or number
			const { lines, lineNumber, columnNumber } = getLineInfo();
			const currentLine = lines[lineNumber];
			let repeat = getRepeat(currentLine);
			const original = repeat;

			const num = startsWithNumberAndPeriod(repeat);
			// line starts with number and period? - increment
			if (num) repeat = `${num + 1}. `;

			if (repeat && original.length < columnNumber) {
				e.preventDefault();
				if (num) correctFollowing(lineNumber);
				await addContent({
					display: "inline",
					text: `\n${repeat}`,
					icon: "",
					title: "",
				});
			} else if (repeat && original.length === columnNumber) {
				// remove if the repeat and caret at the end of the original
				e.preventDefault();
				const selectionEnd = textArea.selectionEnd;
				const newPos = selectionEnd - original.length;
				for (let i = 0; i < original.length; i++) {
					valueTextarea = removeChar(
						valueTextarea,
						textArea.selectionEnd - (i + 1),
					);
				}
				setTimeout(async () => {
					textArea.setSelectionRange(newPos, newPos);
					textArea.focus();
					await addContent({
						display: "inline",
						text: `\n`,
						icon: "",
						title: "",
					});
				}, 0);
			}
		} else {
			const nextCharIsClosing = Object.values(keyPairs).includes(nextChar);
			const highlighted = textArea.selectionStart !== textArea.selectionEnd;
			if (e.ctrlKey && e.key) {
				// keyboard shortcut
				const matchedEl = contentElements.find((el) => el.key === e.key);
				if (matchedEl) addContent(matchedEl);
			} else if (
				nextCharIsClosing &&
				(nextChar === e.key || e.key === "ArrowRight") &&
				openChars.length &&
				!highlighted
			) {
				// type over the next character instead of inserting
				e.preventDefault();
				textArea.setSelectionRange(
					textArea.selectionStart + 1,
					textArea.selectionEnd + 1,
				);
				openChars.pop();
			} else if (e.key in keyPairs) {
				e.preventDefault();
				await addContent({
					display: "wrap",
					text: e.key,
					icon: "",
					title: "",
				});
				openChars.push(e.key);
			}
		}
	};

	/**
	 * - trims the selection if there is an extra space around it
	 */
	const trimSelection = () => {
		if (textArea.selectionStart !== textArea.selectionEnd) {
			if (valueTextarea[textArea.selectionStart] === " ") {
				textArea.setSelectionRange(
					textArea.selectionStart + 1,
					textArea.selectionEnd,
				);
			}
			if (valueTextarea[textArea.selectionEnd - 1] === " ") {
				textArea.setSelectionRange(
					textArea.selectionStart,
					textArea.selectionEnd - 1,
				);
			}
		}
	};

	const updateSelectionStart = () => {
		selectionStartTextarea = textArea.selectionStart;
	};

	/**
	 * @returns lines as an array, current line number, current column number
	 *
	 * @example
	 *
	 * ```js
	 * const { lines, lineNumber, columnNumber } = getLineInfo();
	 * ```
	 */
	const getLineInfo = () => {
		const lines = valueTextarea.split("\n");
		let characterCount = 0;
		for (let i = 0; i < lines.length; i++) {
			// for each line
			characterCount++; // account for removed "\n" due to .split()
			characterCount += lines[i].length;
			// find the line that the cursor is on
			if (characterCount > textArea.selectionEnd) {
				return {
					lines,
					lineNumber: i,
					columnNumber:
						textArea.selectionEnd - (characterCount - lines[i].length - 1),
				};
			}
		}
		return { lines, lineNumber: 0, columnNumber: 0 };
	};

	/**
	 * - splits the content by "```" and finds the current index
	 * of the selectionStart
	 *
	 * @returns current block (index) of selectionStart
	 */
	const getCurrentBlock = () => {
		const blocks = valueTextarea.split("```");
		let totalChars = 0;
		for (const [i, block] of blocks.entries()) {
			totalChars += block.length + 3;
			if (textArea.selectionStart < totalChars) {
				return i;
			}
		}
		return 0;
	};

	/**
	 * - checks if there is a block element or a number
	 * at the beginning of the string
	 *
	 * @param str
	 * @returns what is found, or the empty string
	 */
	const getRepeat = (str: string) => {
		const blockStrings: string[] = [];
		contentElements.forEach((el) => {
			if (el.display === "block") blockStrings.push(el.text);
		});
		for (let i = 0; i < blockStrings.length; i++) {
			const repeatString = blockStrings[i];
			if (str.startsWith(repeatString)) {
				return repeatString;
			}
		}
		const repeatNum = startsWithNumberAndPeriod(str);
		if (repeatNum) return `${repeatNum}. `;
		return "";
	};

	/**
	 * @param str
	 * @returns the number, if the string starts with a number and a period
	 */
	const startsWithNumberAndPeriod = (str: string) => {
		const result = str.match(/^(\d+)\./);
		return result ? Number(result[1]) : null;
	};

	/**
	 * - increments/decrements the start of following lines if they are numbers
	 *
	 * ### prevents this:
	 *
	 * ```
	 * 1. presses enter here when two items in list
	 * 2.
	 * 2.
	 * ```
	 *
	 * ### instead:
	 *
	 * ```
	 * 1.
	 * 2.
	 * 3.
	 * ```
	 *
	 * @param currentLineNumber
	 * @param decrement if following lines should be decremented instead of incremented
	 */
	const correctFollowing = (currentLineNumber: number, decrement = false) => {
		const { lines } = getLineInfo();
		for (let i = currentLineNumber + 1; i < lines.length; i++) {
			const num = startsWithNumberAndPeriod(lines[i]);
			if (!num) {
				break;
			} else {
				let newNum: number;
				if (decrement) {
					if (num > 1) {
						newNum = num - 1;
					} else {
						break;
					}
				} else {
					newNum = num + 1;
				}
				lines[i] = lines[i].slice(String(num).length); // remove number from start
				lines[i] = String(newNum) + lines[i];
			}
		}
		valueTextarea = lines.join("\n");
	};

	onMount(() => (clientJs = true));
</script>

<textarea
	id={idTextarea}
	class={classTextarea}
	name={nameTextarea}
	placeholder={placeholderTextarea}
	on:keydown={onKeyDown}
	on:keyup={updateSelectionStart}
	on:dblclick={trimSelection}
	bind:value={valueTextarea}
	bind:this={textArea}
	on:click={() => {
		openChars = [];
		updateSelectionStart();
	}}
	on:input
/>

<div id={idControls} class={classControls}>
	{#each contentElements as el}
		<button
			type="button"
			class={el.class ? `${classButton} ${el.class}` : classButton}
			on:click={() => addContent(el)}
			title={el.title}
			disabled={!clientJs}
		>
			{#if typeof el.icon !== "string"}
				<svelte:component this={el.icon} />
			{:else}
				{el.icon}
			{/if}
		</button>
	{/each}
</div>
