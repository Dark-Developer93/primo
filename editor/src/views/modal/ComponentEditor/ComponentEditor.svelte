<script context="module">
  import {writable, get} from 'svelte/store'

  const leftPaneSize = writable(get(onMobile) ? '100%' : '50%');
  const rightPaneSize = writable('50%');
  const topPaneSize = writable(get(onMobile) ? '100%' : '50%');
  const bottomPaneSize = writable('50%');
  const orientation = writable('horizontal')
  const activeTab = writable(0)
</script>

<script lang="ts">
  import {_ as C} from 'svelte-i18n'
  import { cloneDeep, find, isEqual, chain as _chain} from 'lodash-es';
  import HSplitPane from './HSplitPane.svelte';
  import { getPlaceholderValue, getEmptyValue } from '../../../utils';
  import ModalHeader from '../ModalHeader.svelte';
  import { Tabs, Card } from '../../../components/misc';
  import FullCodeEditor from './FullCodeEditor.svelte';
  import { CodePreview } from '../../../components/misc';
  import GenericFields from '../../../components/GenericFields.svelte'
  import {autoRefresh} from '../../../components/misc/CodePreview.svelte'

  import {
    processCode,
    processCSS,
    wrapInStyleTags,
  } from '../../../utils';
  import { locale, onMobile } from '../../../stores/app/misc';

  import { content, code as siteCode } from '../../../stores/data/draft';
  import {
    id as pageID,
    code as pageCode
  } from '../../../stores/app/activePage';
  import { showingIDE } from '../../../stores/app';
  import { Component } from '../../../const';
  import type { Component as ComponentType, Symbol as SymbolType, Field as FieldType } from '../../../const';
  import { getPageData } from '../../../stores/helpers';
  import { tick } from 'svelte';

  export let component:ComponentType|SymbolType = Component();
  export let header = {
    label: 'Create Component',
    icon: 'fas fa-code',
    button: {
      icon: 'fas fa-plus',
      label: 'Add to page',
      onclick: (component) => {
        console.warn('Component not going anywhere', component);
      },
    },
  };

  const placeholders = new Map()
  function getCachedPlaceholder(field) {
    const key = JSON.stringify(field)
    if (placeholders.has(key)) {
      return placeholders.get(key)
    } else {
      const val = getPlaceholderValue(field)
      placeholders.set(key, val)
      return val
    }
  }

  let localComponent:SymbolType = cloneDeep(component) // local copy of component to modify & save 
  let localContent = component.type === 'symbol' ? {} : getComponentContent($content) // local copy of component content to modify & save

  // component data w/ page/site data included (for compilation)
  $: data = {
    ...getPageData({ loc: $locale }),
    ...getSymbolPlaceholders(fields),
    ...(localContent[$locale])
  }

  function getSymbolPlaceholders(fields) {
    return _chain(fields).keyBy('key').mapValues(f => f.default || getCachedPlaceholder(f)).value()
  }

  // parse component-specific content out of site content tree (keeping separate locales)
  function getComponentContent(siteContent): object {
    return _chain(Object.entries(siteContent)) 
      .map(([locale]) => ({
        locale,
        content: $content[locale][$pageID]?.[localComponent.id] || getSymbolPlaceholders(localComponent.fields)
      }))
      .keyBy('locale')
      .mapValues('content')
      .value()
  }
  
  $: setupComponent($locale) // swap content out of on-screen fields
  function setupComponent(loc) {
    fields = getFieldValues(fields, loc)
  }

  // hydrate fields with content (placeholder if passed component is a Symbol)
  function getFieldValues(fields:Array<FieldType>, loc:string): Array<any> {
    return fields.map(field => {
      if (component.type === 'symbol') {
        return ({
          ...field,
          value: getCachedPlaceholder(field)
        })
      } else {
        const field_value = localContent[loc]?.[field.key]
        const value = (field_value !== undefined ? field_value : getCachedPlaceholder(field))
        return ({
          ...field,
          value
        })
      }
    })
  }


  // Ensure all content keys match field keys
  $: component.type !== 'symbol' && syncFieldKeys(fields) 
  $: component.type !== 'symbol' && syncLocales($content)

  function syncLocales(content) {
    // runs when adding new locale from ComponentEditor
    Object.keys(content).forEach((loc) => {
      if (!localContent[loc]) {
        localContent = {
          ...localContent,
          [loc]: localContent['en']
        }
      }
    })
  }

  function syncFieldKeys(fields) {
    removeNonexistantKeys() // delete keys from content that do not appear in fields
    addMissingKeys() // add keys that do appear in fields

    function addMissingKeys() {
      let updatedContent = cloneDeep(localContent)
      fields.forEach(field => {
        if (localContent[$locale][field.key] === undefined) {
          Object.keys(localContent).forEach(loc => {
            updatedContent[loc][field.key] = getEmptyValue(field)
          })
        }
      })
      localContent = updatedContent
    }

    // Remove content when field deleted
    function removeNonexistantKeys() {
      let updatedContent = cloneDeep(localContent)
      Object.keys(localContent[$locale]).forEach((key) => {
        if (!find(fields, ['key', key])) {
          Object.keys(localContent).forEach(loc => {
            delete updatedContent[loc][key]
          })
        } 
      })
      localContent = updatedContent
      refreshPreview()
    }
  }


  function saveLocalContent(): void {
    localContent = {
      ...localContent,
      [$locale]: {
        ...localContent[$locale],
        ..._chain(fields).keyBy('key').mapValues('value').value()
      }
    }
  }

  function saveLocalValue(property:'html'|'css'|'js'|'fields', value:any): void {
    if (property === 'fields') {
      localComponent.fields = value
      fields = getFieldValues(value, $locale)
    } else {
      localComponent.code[property] = value
    }
  }

  let loading = false;

  // raw code bound to code editor
  let rawHTML = localComponent.code.html;
  let rawCSS = localComponent.code.css;
  let rawJS = localComponent.code.js;

  // changing codes triggers compilation
  $: $autoRefresh && compileComponentCode({
    html: rawHTML,
    css: rawCSS,
    js: rawJS
  });

  // on-screen fields
  let fields = localComponent.fields;

  let componentApp; // holds compiled component
  let compilationError; // holds compilation error


  // ensure placeholder values always conform to form
  // TODO: do for remaining fields
  $: fields = fields.map(field => {
    if (component.type === 'symbol' && field.type === 'link' && !field.value?.url) return {
      ...field,
      value: getCachedPlaceholder(field) 
    }
    else return field 
  })



  // hover preview node, highlight line in code editor

  // hover tag in code editor, highlight preview node


  let disableSave = false;
  async function compileComponentCode({ html, css, js }) {
    disableSave = true;
    loading = true

    await compile();
    disableSave = compilationError
    await setTimeout(() => {
      loading = false
    }, 200)

    async function compile() {
      const parentCSS = await processCSS($siteCode.css + $pageCode.css)
      const res = await processCode({
        code: {
          html: `${html}
      <svelte:head>
        ${$pageCode.html.head}
        ${$siteCode.html.head}
        ${wrapInStyleTags(parentCSS)}
      </svelte:head>
      ${$pageCode.html.below}
      ${$siteCode.html.below}
      `,
          css,
          js,
        },
        data,
        buildStatic: false,
      });
      compilationError = res.error;
      componentApp = res.js;
      saveLocalValue('html', html);
      saveLocalValue('css', css);
      saveLocalValue('js', js);
    }
  }

  const tabs = [
    {
      id: 'code',
      label: $C('Code'),
      icon: 'code',
    },
    {
      id: 'fields',
      label: $C('Fields'),
      icon: 'database',
    },
  ];

  let previewUpToDate = false
  $: rawHTML, rawCSS, rawJS, previewUpToDate = false // reset when code changes

  async function refreshPreview() {
    await compileComponentCode({
      html: rawHTML,
      css: rawCSS,
      js: rawJS
    })
    previewUpToDate = true
  }

  async function saveComponent() {

    if (!previewUpToDate) {
      await refreshPreview()
    }

    const ExtractedComponent = (component) => ({
      ...component,
      content: localContent,
      fields: fields.map(field => {
        delete field.value
        return field
      })
    })

    if (!disableSave) {
      const component = ExtractedComponent(localComponent)
      header.button.onclick(component);
    }
  }

</script>

<ModalHeader
  {...header}
  warn={() => {
    if (!isEqual(localComponent, component)) {
      const proceed = window.confirm('Undrafted changes will be lost. Continue?');
      return proceed;
    } else return true;
  }}
  button={{ ...header.button, onclick: saveComponent, disabled: disableSave }}>
</ModalHeader>

<main class:showing-ide={$showingIDE} class:showing-cms={!$showingIDE}>
  <HSplitPane
    orientation={$orientation}
    bind:leftPaneSize={$leftPaneSize}
    bind:rightPaneSize={$rightPaneSize}
    bind:topPaneSize={$topPaneSize}
    bind:bottomPaneSize={$bottomPaneSize}
    hideRightPanel={$onMobile}
    hideLeftOverflow={$showingIDE && $activeTab === 0}>
    <div slot="left" lang={$locale}>
      {#if $showingIDE}
        <Tabs {tabs} bind:activeTab={$activeTab} />
        {#if $activeTab === 0}
          <FullCodeEditor
            variants="flex-1"
            bind:html={rawHTML}
            bind:css={rawCSS}
            bind:js={rawJS}
            on:save={saveComponent} 
            on:refresh={refreshPreview}
          />
        {:else if $activeTab === 1}
          <GenericFields bind:fields on:input={refreshPreview} on:delete={async () => {
            await tick() // wait for fields to update
            saveLocalContent()
            refreshPreview()
          }} showCode={true} />
        {/if}
      {:else}
        <GenericFields bind:fields on:save={saveComponent} on:input={() => {
          fields = fields.filter(Boolean) // to trigger setting `data`
          saveLocalContent()
        }} showCode={false} />
      {/if}
    </div>
    <div slot="right">
      <CodePreview
        bind:orientation={$orientation}
        view="small"
        {loading}
        {componentApp}
        {data}
        error={compilationError}
      />
    </div>
  </HSplitPane>
</main>

<style lang="postcss">
  main {
    display: flex; /* to help w/ positioning child items in code view */
    background: var(--primo-color-black);
    color: var(--color-gray-2);
    padding: 0 0.5rem;
    flex: 1;
    overflow: hidden;

    --PrimaryButton-bg: var(--color-gray-8);
    --PrimaryButton-bg-hover: var(--color-gray-9);
  }

  [slot="right"] {
    width: 100%;
  }

  :global(.showing-cms [slot="left"]) {
    height: 100% !important;
  }

  :global(.showing-cms .wrapper.vertical) {
    height: 100% !important;
  }

  [slot="left"] {
    height: calc(100% - 45px);

    display: flex;
    flex-direction: column;
  }

</style>
