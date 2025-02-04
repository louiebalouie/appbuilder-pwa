<!--
@component
A component for displaying scripture.  
TODO:
- find a way to scroll smoothly, as CSS only option does not work as expected.
- save graft info so that references can be handled
- parse introduction for references
-->
<script lang="ts">
    import { onDestroy } from 'svelte';
    import { Proskomma } from 'proskomma-core';
    import { SofriaRenderFromProskomma } from 'proskomma-json-tools';
    import { thaw } from '../scripts/thaw';
    import {
        audioHighlight,
        refs,
        scrolls,
        audioActive,
        mainScroll,
        bodyFontSize,
        bodyLineHeight,
        themeColors
    } from '$lib/data/stores';
    import { onClickText, deselectAllElements } from '$lib/scripts/verseSelectUtil';

    import { LoadingIcon } from '$lib/icons';

    const pk = new Proskomma();
    let container: HTMLElement;
    const seprgx = /(\.|\?|!|:|;|,|')/g;

    /**unique key to use for groupStore modifier*/
    const key = {};

    let group = 'default';
    let scrollId: string;
    let scrollMod: any;
    const unSub = scrolls.subscribe((val, mod) => {
        scrollId = val;
        scrollMod = mod;
    }, group);

    /**scrolls element with id into view*/
    const scrollTo = (id: string) => {
        if (scrollMod === key) return;
        container
            ?.querySelector(
                `div[data-verse="${id.split('-')[0]}"][data-phrase="${id.split('-')[1]}"]`
            )
            ?.scrollIntoView();
    };
    $: scrollTo(scrollId);

    const onlySpaces = (str) => {
        return str.trim().length === 0;
    };

    let lastVerseInView = '';
    let displayingIntroduction = false;

    const handleScroll = (() => {
        let scrollTimer: NodeJS.Timeout;

        return (trigger) => {
            clearTimeout(scrollTimer);
            scrollTimer = setTimeout(() => {
                const items = Array.from(container?.getElementsByClassName('scroll-item'))
                    .filter((it, i) => {
                        const rect = it.getBoundingClientRect();
                        const win = container.getBoundingClientRect();

                        return (
                            rect.top - win.top >= $mainScroll.top &&
                            rect.bottom - win.top <= $mainScroll.height + $mainScroll.top
                        );
                    })
                    .map(
                        (el) => `${el.getAttribute('data-verse')}-${el.getAttribute('data-phrase')}`
                    );

                scrolls.set(items[0], group, key);
                lastVerseInView = items.pop();
            }, 500);
        };
    })();
    $: handleScroll([$mainScroll, $refs]);

    /**updates highlight*/
    const updateHighlight = (h: string, color: string) => {
        const a = h.split(',');
        let el = container?.getElementsByClassName('highlighting')?.item(0);
        let node = el?.getAttributeNode('style');
        el?.removeAttributeNode(node);
        el?.classList.remove('highlighting');
        if (!$audioActive || a[0] !== $refs.docSet || a[1] !== $refs.book || a[2] !== $refs.chapter)
            return;
        el = container?.querySelector(`div[data-verse="${a[3]}"][data-phrase="${a[4]}"]`);
        el?.setAttribute('style', 'background-color: ' + color + ';');
        el?.classList.add('highlighting');
        if (
            `${el?.getAttribute('data-verse')}-${el?.getAttribute('data-phrase')}` ===
            lastVerseInView
        )
            el?.scrollIntoView();
    };
    $: updateHighlight($audioHighlight, highlightColor);

    const countSubheadingPrefixes = (subHeadings: [string], labelPrefix: string) => {
        let result = 0;
        for (let i in subHeadings) {
            if (subHeadings[i] === labelPrefix) {
                result++;
            }
        }
        return result;
    };
    const parsePhrase = (inner) => {
        let phrases = inner.split(seprgx);
        for (let i = 1; i < phrases.length; i += 2) {
            phrases[i - 1] += phrases[i];
        }
        phrases = phrases.filter((s) => s.length > 0 && (s.length > 1 || !s.match(seprgx)));
        //move chars orphaned by phrase parsing to preceding phrase
        if (phrases.length > 1) {
            for (let i = 0; i < phrases.length - 1; i++) {
                const next = phrases[i + 1].split('');
                let c = next.shift();
                while (c && c.match(/[^_a-z]/i)) {
                    phrases[i] += c;
                    c = next.shift();
                }
                if (c) {
                    next.unshift(c);
                    phrases[i + 1] = next.join('');
                } else {
                    phrases.splice(i + 1, 1);
                    i--;
                }
            }
        }
        // console.log('parsePhrase %o %o', inner, phrases);
        return phrases;
    };

    const startPhrase = (workspace, indexOption = 'advance') => {
        // console.log('Start phrase!!!');
        const fnc = 'abcdefghijklmnopqrstuvwxyz';
        const div = document.createElement('div');
        if (!workspace.introductionGraft) {
            switch (indexOption) {
                case 'reset':
                    workspace.currentPhraseIndex = 0;
                    break;
                case 'advance':
                    workspace.currentPhraseIndex++;
                    break;
                default:
                    break;
            }
            const phraseIndex = fnc.charAt(workspace.currentPhraseIndex);
            div.id = workspace.currentVerse + phraseIndex;
            div.setAttribute('data-verse', workspace.currentVerse);
            div.setAttribute('data-phrase', phraseIndex);
            div.classList.add('txs', 'seltxt', 'scroll-item');
        } else {
            div.id = '+' + parseInt(workspace.introductionIndex);
            div.classList.add('txs');
            workspace.introductionIndex++;
        }
        return div.cloneNode(true);
    };
    const addText = (workspace, append = true) => {
        if (!onlySpaces(workspace.text)) {
            let phrases = [];
            if (!workspace.introductionGraft) {
                phrases = parsePhrase(workspace.text);
            } else {
                // Don't parse introduction text.  Each paragraph
                // is a single div.
                phrases[0] = workspace.text;
            }
            for (let i = 0; i < phrases.length; i++) {
                const div = workspace.phraseDiv.cloneNode(true);
                const phrase = phrases[i];
                const textNode = document.createTextNode(phrase);
                div.appendChild(textNode);
                if (append || i < phrases.length - 1) {
                    workspace.paragraphDiv.appendChild(div.cloneNode(true));
                    workspace.phraseDiv = startPhrase(workspace);
                } else {
                    workspace.phraseDiv = div;
                }
            }
        }
        workspace.text = '';
        return;
    };

    const processText = (introductionGraft, showIntroduction, titleGraft) => {
        let returnValue = false;
        if (introductionGraft == showIntroduction || (titleGraft && showIntroduction)) {
            returnValue = true;
        }
        return returnValue;
    };
    let bookRoot = document.createElement('div');
    // console.log('START: %o', bookRoot);
    let loading = true;

    const output = {};
    const query = async (docSet: string, bookCode: string, chapter: string) => {
        // console.log('PARMS: bc: %o, chapter: %o, collection: %o', bookCode, chapter, docSet);
        const docslist = await pk.gqlQuery('{docSets { id } }');
        // console.log('LIST %o', docslist);
        // console.log('Displaying Introduction %o', displayingIntroduction);
        let found = false;
        for (const doc of docslist.data.docSets) {
            // console.log('ID: %o', doc.id);
            if (doc.id === docSet) {
                // console.log('Found');
                found = true;
                break;
            }
        }

        if (!found) {
            // console.log('fetch %o pkf', docSet);
            const res = await fetch(`collections/${docSet}.pkf`).then((r) => {
                return r.arrayBuffer();
            });
            if (res.byteLength) {
                // console.log('awaiting thaw');
                const uint8res = new Uint8Array(res);
                thaw(pk, uint8res);
            }
        }

        const fnc = 'abcdefghijklmnopqrstuvwxyz';
        const cl = new SofriaRenderFromProskomma({
            proskomma: pk,
            actions: {
                startDocument: [
                    {
                        description: 'Set up; Book heading',
                        test: () => true,
                        action: ({ context, workspace }) => {
                            /* console.log(
                                'Start Document: %o, %o',
                                context,
                                context.document.metadata.document
                            );*/
                            bookRoot.replaceChildren();
                            workspace.root = bookRoot;
                            workspace.text = '';
                            workspace.footnoteIndex = 0;
                            workspace.introductionIndex = 0;
                            workspace.firstVerse = true;
                            workspace.currentVerse = 'none';
                            workspace.currentPhraseIndex = 0;
                            workspace.introductionGraft = false;
                            workspace.titleGraft = false;
                            workspace.paragraphDiv = document.createElement('div');
                            workspace.titleBlockDiv = document.createElement('div');
                            workspace.phraseDiv = document.createElement('div');
                            workspace.subheaders = [];
                            deselectAllElements();
                        }
                    }
                ],
                startParagraph: [
                    {
                        description: 'Start HTML para with appropriate class',
                        test: () => true,
                        action: ({ context, workspace }) => {
                            /* console.log(
                                'Start Paragraph %o %o',
                                context.sequences[0].block,
                                context.sequences[0].type
                            ); */
                            const sequenceType = context.sequences[0].type;
                            if (
                                processText(
                                    workspace.introductionGraft,
                                    displayingIntroduction,
                                    workspace.titleGraft
                                )
                            ) {
                                var paraClass =
                                    context.sequences[0].block.subType.split(':')[1] ||
                                    context.sequences[0].block.subType;
                                if (sequenceType === 'main' && !displayingIntroduction) {
                                    if (workspace.firstVerse == true) {
                                        paraClass = 'm';
                                        workspace.firstVerse = false;
                                    }
                                    workspace.paragraphDiv = document.createElement('div');
                                    workspace.paragraphDiv.classList.add(paraClass);
                                    if (workspace.currentVerse != 'none') {
                                        workspace.phraseDiv = startPhrase(workspace, 'keep');
                                        /* console.log(
                                            'Paragraph Start phrase: %o',
                                            workspace.phraseDiv
                                        );*/
                                    }
                                } else if (sequenceType == 'introduction') {
                                    workspace.paragraphDiv = document.createElement('div');
                                    workspace.paragraphDiv.classList.add(paraClass);
                                    if (workspace.introductionIndex == 1) {
                                        workspace.phraseDiv = startPhrase(workspace, 'keep');
                                    }
                                } else {
                                    //NO workspace.htmlBits.push(`<div class="${paraClass}">`)
                                }
                            }
                        }
                    }
                ],
                endParagraph: [
                    {
                        description: 'End HTML para',
                        test: () => true,
                        action: ({ context, workspace }) => {
                            const sequenceType = context.sequences[0].type;
                            // console.log('End paragraph: Sequence type ' + sequenceType);
                            /* console.log(
                                'End Paragraph %o %o',
                                context.sequences[0].block,
                                context.sequences[0].type
                            );*/
                            if (
                                processText(
                                    workspace.introductionGraft,
                                    displayingIntroduction,
                                    workspace.titleGraft
                                )
                            ) {
                                if (sequenceType == 'main') {
                                    // Build div
                                    // console.log('End Paragaph text:' + workspace.text);
                                    if (workspace.currentVerse != 'none') {
                                        // console.log('END PHRASE: P');
                                        // console.log('PHRASE: ' + workspace.text);
                                        addText(workspace);
                                        // console.log('OUT: %o ', workspace.paragraphDiv);
                                    }
                                    workspace.root.appendChild(workspace.paragraphDiv);
                                } else if (sequenceType == 'title') {
                                    // console.log('End paragraph title text:' + workspace.text);
                                    const div = document.createElement('div');
                                    var paraClass =
                                        context.sequences[0].block.subType.split(':')[1] ||
                                        context.sequences[0].block.subType;
                                    div.classList.add(paraClass);
                                    const span = document.createElement('span');
                                    span.classList.add(paraClass);
                                    span.innerText = workspace.text;
                                    div.appendChild(span);
                                    workspace.titleBlockDiv.appendChild(div);
                                    workspace.text = '';
                                } else if (sequenceType == 'heading') {
                                    // console.log('Header text: ' + workspace.text);
                                    const div = document.createElement('div');
                                    var headingParaClass =
                                        context.sequences[0].block.subType.split(':')[1] ||
                                        context.sequences[0].block.subType;
                                    div.classList.add(headingParaClass);
                                    const prefix = headingParaClass.replaceAll(/[0-9]/g, '');
                                    workspace.subheaders.push(prefix);
                                    const count = countSubheadingPrefixes(
                                        workspace.subheaders,
                                        prefix
                                    );
                                    const innerDiv = document.createElement('div');
                                    innerDiv.id = prefix + count;
                                    innerDiv.innerText = workspace.text;
                                    div.appendChild(innerDiv);
                                    workspace.root.appendChild(div);
                                    workspace.text = '';
                                } else if (sequenceType == 'introduction') {
                                    // console.log('End Paragaph text:' + workspace.text);
                                    addText(workspace);
                                    workspace.root.appendChild(workspace.paragraphDiv);
                                }
                            }
                        }
                    }
                ],
                text: [
                    {
                        description: 'Output text',
                        test: () => true,
                        action: ({ context, workspace }) => {
                            /* console.log(
                                'Text element: %o %o %o',
                                context.sequences[0].element.type,
                                context.sequences[0].element.text,
                                context.sequences[0].block
                            );*/
                            if (
                                processText(
                                    workspace.introductionGraft,
                                    displayingIntroduction,
                                    workspace.titleGraft
                                )
                            ) {
                                const blockType = context.sequences[0].block.subType;
                                if (blockType.includes('usfm:x') || blockType.includes('usfm:f')) {
                                    // Graft Text
                                } else if (blockType.includes('usfm:ip')) {
                                    // Introduction
                                    workspace.text += context.sequences[0].element.text;
                                } else {
                                    workspace.text += context.sequences[0].element.text;
                                }
                            }
                        }
                    }
                ],
                mark: [
                    {
                        description: 'Output text',
                        test: () => true,
                        action: ({ context, workspace }) => {
                            const element = context.sequences[0].element;
                            /* console.log(
                                'Mark: SubType %o, Atts: %o',
                                element.subType,
                                element.atts
                            ); */
                            if (
                                processText(
                                    workspace.introductionGraft,
                                    displayingIntroduction,
                                    workspace.titleGraft
                                )
                            ) {
                                if (element.subType === 'chapter_label') {
                                    const div = document.createElement('div');
                                    div.classList.add('c-drop');
                                    div.innerText = element.atts['number'];
                                    workspace.paragraphDiv.appendChild(div);
                                    //                                    workspace.htmlBits.push(`<div class="c-drop"> ${element.atts['number']}</div>\n`);
                                } else if (element.subType === 'verses_label') {
                                    var spanV = document.createElement('span');
                                    spanV.classList.add('v');
                                    spanV.innerText = element.atts['number'];
                                    var spanVsp = document.createElement('span');
                                    spanVsp.classList.add('vsp');
                                    spanVsp.innerText = '\u00A0'; // &nbsp
                                    var div = workspace.phraseDiv.cloneNode(true);
                                    div.appendChild(spanV.cloneNode(true));
                                    div.appendChild(spanVsp.cloneNode(true));
                                    // console.log('OUT: %o %o %o', div, spanV, spanVsp);
                                    workspace.phraseDiv = div.cloneNode(true);
                                }
                            }
                        }
                    }
                ],
                endDocument: [
                    {
                        description: 'Set up',
                        test: () => true,
                        action: ({ context, workspace, output }) => {
                            // console.log('End Document');
                            var els = document.getElementsByTagName('div');
                            for (var i = 0; i < els.length; i++) {
                                if (els[i].classList.contains('seltxt') && els[i].id != '') {
                                    els[i].addEventListener('click', onClickText, false);
                                }
                            }
                        }
                    }
                ],
                startSequence: [
                    {
                        description: 'Start Sequence',
                        test: () => true,
                        action: ({ context, workspace }) => {
                            const sequenceType = context.sequences[0].type;
                            // console.log('start sequence %o', sequenceType);
                            if (
                                processText(
                                    workspace.introductionGraft,
                                    displayingIntroduction,
                                    workspace.titleGraft
                                )
                            ) {
                                if (sequenceType === 'title') {
                                    const div = document.createElement('div');
                                    div.setAttribute('data-verse', 'title');
                                    div.setAttribute('data-phrase', 'none');
                                    div.classList.add('scroll-item');
                                    workspace.titleBlockDiv = div;
                                }
                            }
                        }
                    }
                ],
                endSequence: [
                    {
                        description: 'End Sequence',
                        test: () => true,
                        action: ({ context, workspace }) => {
                            const sequenceType = context.sequences[0].type;
                            // console.log('End sequence |%o|', sequenceType);
                            if (
                                processText(
                                    workspace.introductionGraft,
                                    displayingIntroduction,
                                    workspace.titleGraft
                                )
                            ) {
                                if (sequenceType === 'title') {
                                    const div = workspace.titleBlockDiv;
                                    div.innerHTML += `<div class="b"></div><div class="b"></div>`;
                                    // console.log('TITLE DIV %o', div);
                                    workspace.root.append(div);
                                }
                            }
                        }
                    }
                ],
                blockGraft: [
                    {
                        description: 'Block Graft',
                        test: () => true,
                        action: (environment) => {
                            // console.log('Block Graft %o', environment.context.sequences[0].block);
                            const currentBlock = environment.context.sequences[0].block;
                            const graftRecord = {
                                type: currentBlock.type,
                                sequence: {}
                            };
                            if (currentBlock.subType === 'introduction') {
                                // console.log('*** START INTRODUCTION ***');
                                environment.workspace.introductionGraft = true;
                                environment.workspace.introductionIndex = 1;
                            } else if (currentBlock.subType === 'title') {
                                environment.workspace.titleGraft = true;
                            }
                            if (currentBlock.sequence) {
                                //                                environment.workspace.htmlBits.push(`<span data-graft="${currentBlock.sequence.id}">`);
                                graftRecord.sequence = {};
                                const cachedSequencePointer = environment.workspace.currentSequence;
                                environment.workspace.currentSequence = graftRecord.sequence;
                                environment.context.renderer.renderSequence(environment);
                                environment.workspace.currentSequence = cachedSequencePointer;
                                //                                environment.workspace.htmlBits.push(`</span>`);
                            }
                            if (currentBlock.subType === 'introduction') {
                                // console.log('*** END INTRODUCTION');
                                environment.workspace.introductionGraft = false;
                            } else if (currentBlock.subType === 'title') {
                                environment.workspace.titleGraft = false;
                            }
                            // console.log('Block Graft End %o %o', graftRecord, currentBlock);
                        }
                    }
                ],
                inlineGraft: [
                    {
                        description: 'Inline graft',
                        test: () => true,
                        action: (environment) => {
                            const element = environment.context.sequences[0].element;
                            const workspace = environment.workspace;
                            /* console.log(
                                'Inline Graft Type: %o, Subtype: %o, id: %o %o',
                                element.type,
                                element.subType,
                                element.sequence.id,
                                environment.context.sequences[0].element
                            );*/
                            const graftRecord = {
                                type: element.type,
                                subtype: element.subType,
                                sequence: {}
                            };
                            if (element.subType === 'xref' || element.subType === 'footnote') {
                                // console.log('PHRASE G: ' + workspace.text);
                                addText(workspace, false);
                                const div = workspace.phraseDiv.cloneNode(true);
                                // console.log('Footnote or xref');
                                const span = document.createElement('span');
                                span.setAttribute('data-graft', `X-${workspace.footnoteIndex + 1}`);
                                span.classList.add('footnote');
                                const a = document.createElement('a');
                                const sup = document.createElement('sup');
                                sup.innerHTML = fnc.charAt(workspace.footnoteIndex);
                                a.appendChild(sup);
                                span.appendChild(a);
                                div.appendChild(span);
                                workspace.paragraphDiv.appendChild(div.cloneNode(true));
                                workspace.footnoteIndex++;
                                workspace.phraseDiv = startPhrase(workspace);
                            }
                            const cachedSequencePointer = workspace.currentSequence;
                            workspace.currentSequence = graftRecord.sequence;
                            environment.context.renderer.renderSequence(environment);
                            workspace.currentSequence = cachedSequencePointer;
                            // console.log('Inline Graft End');
                        }
                    }
                ],
                startWrapper: [
                    {
                        description: 'Start Wrapper',
                        test: () => true,
                        action: ({ context, workspace }) => {
                            // console.log('Start Wrapper %o', context.sequences[0].element);
                            let element = context.sequences[0].element;
                            if (element.subType === 'verses') {
                                workspace.currentVerse = element.atts.number;
                                workspace.phraseDiv = startPhrase(workspace, 'reset');
                                // console.log('IN: %o', workspace.phraseDiv);
                            }
                        }
                    }
                ],
                endWrapper: [
                    {
                        description: 'End Wrapper',
                        test: () => true,
                        action: ({ context, workspace }) => {
                            // console.log('End Wrapper %o', context.sequences[0].element);
                            let element = context.sequences[0].element;
                            if (element.subType === 'verses') {
                                if (!onlySpaces(workspace.text)) {
                                    // console.log('PHRASE: ' + workspace.text);
                                    addText(workspace);
                                    // console.log('OUT: %o ', workspace.paragraphDiv);
                                }
                                // console.log('END PHRASE: V');
                                workspace.currentVerse = 'none';
                            }
                        }
                    }
                ],
                startMilestone: [
                    {
                        description: 'Start Milestone',
                        test: () => true,
                        action: ({ context }) => {
                            // console.log('Start Milestone: %o', context.sequences[0].element);
                        }
                    }
                ],
                endMilestone: [
                    {
                        description: 'End Milestone',
                        test: () => true,
                        action: ({ context }) => {
                            // console.log('End Milestone %o', context.sequences[0].element);
                        }
                    }
                ]
            },
            debugLevel: 0
        });
        const docsResult = pk.gqlQuerySync(
            '{documents { docSetId id bookCode: header(id: "bookCode")} }'
        );
        // console.log('docsResult %o', docsResult);
        const bookLookup = {};
        for (const docRecord of docsResult.data.documents) {
            if (docRecord.docSetId === docSet) {
                bookLookup[docRecord.bookCode] = docRecord.id;
            }
        }
        const docId = bookLookup[bookCode];
        cl.renderDocument({ docId, config: { chapters: [chapter] }, output });
        loading = false;
        // console.log('DONE %o', root);
    };

    $: fontSize = $bodyFontSize + 'px';

    $: lineHeight = $bodyLineHeight + '%';
    $: highlightColor = $themeColors['TextHighlightColor'];

    $: (() => {
        let chapterToDisplay = $refs.chapter;
        if (chapterToDisplay == 'i') {
            // console.log('Displaying introduction');
            chapterToDisplay = '1';
            displayingIntroduction = true;
        } else {
            displayingIntroduction = false;
        }
        const bookCode = $refs.book;
        const chapter = chapterToDisplay;
        const docSet = $refs.docSet;
        query(docSet, bookCode, chapter);
    })();
    onDestroy(unSub);
</script>

<article class="container" bind:this={container}>
    {#if loading}
        <LoadingIcon />
    {/if}
    <div
        bind:this={bookRoot}
        class:hidden={loading}
        style:font-size={fontSize}
        style:line-height={lineHeight}
    />
</article>
