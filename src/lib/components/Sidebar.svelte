<!--
@component
The sidebar/drawer.
-->
<script>
    import {
        AccountIcon,
        SearchIcon,
        HistoryIcon,
        BookmarkIcon,
        NoteIcon,
        HighlightIcon,
        ShareIcon,
        SettingsIcon,
        TextAppearanceIcon,
        AboutIcon
    } from '$lib/icons';
    import config from '$lib/data/config';
    import { beforeNavigate } from '$app/navigation';
    import { firebaseConfig } from '$lib/data/firebase-config';
    import { s, t, themeColors, language, languageDefault } from '$lib/data/stores';
    const drawerId = 'sidebar';
    const closeDrawer = () => {
        document.activeElement.blur();
    };
    beforeNavigate(closeDrawer);

    const menuItems = config?.menuItems;
    const showSearch = config.mainFeatures['search'];
    const showHistory = config.mainFeatures['history'];
    const showBookmarks = config.mainFeatures['annotation-bookmarks'];
    const showNotes = config.mainFeatures['annotation-notes'];
    const showHighlights = config.mainFeatures['annotation-highlights'];
    const showShare =
        config.mainFeatures['share-app-link'] ||
        config.mainFeatures['share-download-app-link'] ||
        config.mainFeatures['share-apk-file'] ||
        config.mainFeatures['share-apple-app-link'];
    const showAccount = firebaseConfig && config.mainFeatures['user-accounts'];

    $: textColor = $s['ui.drawer.item.text']['color'];
    $: iconColor = $s['ui.drawer.item.icon']['color'];
    $: contentBackgroundColor = $s['ui.background']['background-color'];
    $: drawerBackgroundColor = $s['ui.drawer']['background-color'];
    $: contentTextColor = $themeColors['TextColor'];
</script>

<div class="dy-drawer dy-drawer-mobile">
    <input id={drawerId} type="checkbox" class="dy-drawer-toggle" />
    <div
        class="dy-drawer-content flex flex-col"
        style:background-color={contentBackgroundColor}
        style:color={contentTextColor}
    >
        <!-- Page content here -->
        <slot />
    </div>
    <div class="dy-drawer-side">
        <label for={drawerId} class="dy-drawer-overlay" />
        <ul
            class="dy-menu p-1  w-3/4 sm:w-80 text-base-content"
            style:background-color={drawerBackgroundColor}
        >
            <!-- Sidebar content here -->
            <a class="fill" href="/">
                <picture>
                    <source srcset="images/nav_drawer@2x.png 2x" />
                    <img src="images/nav_drawer.png" alt="Drawer Header" style="width:auto;" />
                </picture>
            </a>
            {#if showAccount}
                <li>
                    <a href="/account" style:color={textColor}>
                        <AccountIcon color={iconColor} />{$t['Account_Page_Title']}
                    </a>
                </li>
                <div class="dy-divider m-1" />
            {/if}
            {#if showSearch}
                <li>
                    <a href="/search" style:color={textColor}>
                        <SearchIcon color={iconColor} />{$t['Menu_Search']}
                    </a>
                </li>
                <div class="dy-divider m-1" />
            {/if}
            {#if showHistory}
                <li>
                    <a href="/history" style:color={textColor}>
                        <HistoryIcon color={iconColor} />{$t['Menu_History']}
                    </a>
                </li>
            {/if}
            {#if showBookmarks}
                <li>
                    <a href="/bookmarks" style:color={textColor}>
                        <BookmarkIcon color={iconColor} />{$t['Annotation_Bookmarks']}
                    </a>
                </li>
            {/if}
            {#if showNotes}
                <li>
                    <a href="/notes" style:color={textColor}>
                        <NoteIcon color={iconColor} />{$t['Annotation_Notes']}
                    </a>
                </li>
            {/if}
            {#if showHighlights}
                <li>
                    <a href="/highlights" style:color={textColor}>
                        <HighlightIcon color={iconColor} />{$t['Annotation_Highlights']}
                    </a>
                </li>
            {/if}
            {#if showHistory || showBookmarks || showNotes || showHighlights}
                <div class="dy-divider m-1" />
            {/if}
            {#if showShare}
                <li>
                    <a href="/share" style:color={textColor}>
                        <ShareIcon color={iconColor} />{$t['Menu_Share_App']}
                    </a>
                </li>
                <div class="dy-divider m-1" />
            {/if}
            <li>
                <a href="/settings" style:color={textColor}>
                    <SettingsIcon color={iconColor} />{$t['Menu_Settings']}
                </a>
            </li>
            <!-- svelte-ignore a11y-missing-attribute -->
            <!-- svelte-ignore a11y-click-events-have-key-events -->
            <li>
                <a on:click={closeDrawer} style:color={textColor}>
                    <TextAppearanceIcon color={iconColor} />{$t['Menu_Text_Appearance']}
                </a>
            </li>
            <div class="dy-divider m-1" />
            {#if menuItems}
                {#each menuItems as item}
                    <li>
                        <a
                            href={item.link['default']}
                            style:color={textColor}
                            target="_blank"
                            rel="noreferrer"
                        >
                            <picture>
                                <source
                                    srcset="icons/menu-items/{item.images[1]
                                        .file} 2x, icons/menu-items/{item.images[2].file} 3x"
                                />
                                <img
                                    src="icons/menu-items/{item.images[0].file}"
                                    height="24"
                                    width="24"
                                    alt={item.title[$language] || item.title[languageDefault]}
                                />
                            </picture>{item.title[$language] || item.title[languageDefault]}
                        </a>
                    </li>
                {/each}
            {/if}
            <li>
                <a href="/about" style:color={textColor}>
                    <AboutIcon color={iconColor} />{$t['Menu_About']}
                </a>
            </li>
        </ul>
    </div>
</div>

<style>
    a {
        text-decoration: none;
    }
    .fill {
        display: flex;
        justify-content: center;
        align-items: center;
        overflow: hidden;
    }
    .fill img {
        flex-shrink: 0;
        min-width: 100%;
        min-height: 100%;
    }
</style>
