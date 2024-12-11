<template></template>
<script>
import remarkParse from 'remark-parse'
import remarkRehype from 'remark-rehype'
import { unified } from 'unified'
import { visit } from 'unist-util-visit'
import { VFile } from 'vfile'
import { urlAttributes } from 'html-url-attributes'
import { toJsxRuntime } from 'hast-util-to-jsx-runtime'
import { Fragment, jsx, jsxs } from 'vue/jsx-runtime'

const unreachable = (message) => {
  throw new Error(message)
}

/** @type {PluggableList} */
const emptyPlugins = []
/** @type {Readonly<RemarkRehypeOptions>} */
const emptyRemarkRehypeOptions = { allowDangerousHtml: true }
const safeProtocol = /^(https?|ircs?|mailto|xmpp)$/i

// Mutable because we `delete` any time it’s used and a message is sent.
/** @type {ReadonlyArray<Readonly<Deprecation>>} */
const deprecations = [
  { from: 'astPlugins', id: 'remove-buggy-html-in-markdown-parser' },
  { from: 'allowDangerousHtml', id: 'remove-buggy-html-in-markdown-parser' },
  {
    from: 'allowNode',
    id: 'replace-allownode-allowedtypes-and-disallowedtypes',
    to: 'allowElement',
  },
  {
    from: 'allowedTypes',
    id: 'replace-allownode-allowedtypes-and-disallowedtypes',
    to: 'allowedElements',
  },
  {
    from: 'disallowedTypes',
    id: 'replace-allownode-allowedtypes-and-disallowedtypes',
    to: 'disallowedElements',
  },
  { from: 'escapeHtml', id: 'remove-buggy-html-in-markdown-parser' },
  { from: 'includeElementIndex', id: '#remove-includeelementindex' },
  {
    from: 'includeNodeIndex',
    id: 'change-includenodeindex-to-includeelementindex',
  },
  { from: 'linkTarget', id: 'remove-linktarget' },
  {
    from: 'plugins',
    id: 'change-plugins-to-remarkplugins',
    to: 'remarkPlugins',
  },
  { from: 'rawSourcePos', id: '#remove-rawsourcepos' },
  { from: 'renderers', id: 'change-renderers-to-components', to: 'components' },
  { from: 'source', id: 'change-source-to-children', to: 'children' },
  { from: 'sourcePos', id: '#remove-sourcepos' },
  { from: 'transformImageUri', id: '#add-urltransform', to: 'urlTransform' },
  { from: 'transformLinkUri', id: '#add-urltransform', to: 'urlTransform' },
]
export default {
  name: 'VueMarkdown',
  props: {
    content: {
      type: String,
      required: true,
    },
    className: {
      type: String,
      default: '',
    },
    remarkPlugins: {
      type: Array,
      default: () => [],
    },
    rehypePlugins: {
      type: Array,
      default: () => [],
    },
    remarkRehypeOptions: {
      type: Object,
      default: () => ({}),
    },
    allowedElements: {
      type: Array,
    },
    disallowedElements: {
      type: Array,
    },
    components: {
      type: Object,
      default: () => ({}),
    },
    skipHtml: {
      type: Boolean,
      default: false,
    },
    unwrapDisallowed: {
      type: Boolean,
      default: false,
    },
    urlTransform: {
      type: Function,
      default: defaultUrlTransform,
    },
    allowElement: {
      type: Function,
      default: null,
    },
  },
  data() {
    return {
      msg: 'Welcome to Your Vue.js App',
      processor: null,
    }
  },
  setup(options, context) {
    const allowedElements = options.allowedElements
    const allowElement = options.allowElement
    const children = options.content || ''
    const className = options.className
    const components = options.components
    const disallowedElements = options.disallowedElements
    const rehypePlugins = options.rehypePlugins || emptyPlugins
    const remarkPlugins = options.remarkPlugins || emptyPlugins
    const remarkRehypeOptions = options.remarkRehypeOptions
      ? { ...options.remarkRehypeOptions, ...emptyRemarkRehypeOptions }
      : emptyRemarkRehypeOptions
    const skipHtml = options.skipHtml
    const unwrapDisallowed = options.unwrapDisallowed
    const urlTransform = options.urlTransform || defaultUrlTransform

    const processor = unified()
      .use(remarkParse)
      .use(remarkPlugins)
      .use(remarkRehype, remarkRehypeOptions)
      .use(rehypePlugins)

    const file = new VFile()

    if (typeof children === 'string') {
      file.value = children
    } else {
      unreachable(
        'Unexpected value `' +
          children +
          '` for `children` prop, expected `string`'
      )
    }

    if (allowedElements && disallowedElements) {
      unreachable(
        'Unexpected combined `allowedElements` and `disallowedElements`, expected one or the other'
      )
    }

    for (const deprecation of deprecations) {
      if (Object.hasOwn(options, deprecation.from)) {
        unreachable(
          'Unexpected `' +
            deprecation.from +
            '` prop, ' +
            (deprecation.to
              ? 'use `' + deprecation.to + '` instead'
              : 'remove it') +
            ' (see <' +
            changelog +
            '#' +
            deprecation.id +
            '> for more info)'
        )
      }
    }

    const mdastTree = processor.parse(file)
    /** @type {Nodes} */
    let hastTree = processor.runSync(mdastTree, file)

    // Wrap in `div` if there’s a class name.
    if (className) {
      hastTree = {
        type: 'element',
        tagName: 'div',
        properties: { className },
        // Assume no doctypes.
        children: /** @type {Array<ElementContent>} */ (
          hastTree.type === 'root' ? hastTree.children : [hastTree]
        ),
      }
    }

    visit(hastTree, transform)

    function transform(node, index, parent) {
      if (node.type === 'raw' && parent && typeof index === 'number') {
        if (skipHtml) {
          parent.children.splice(index, 1)
        } else {
          parent.children[index] = { type: 'text', value: node.value }
        }

        return index
      }

      if (node.type === 'element') {
        /** @type {string} */
        let key

        for (key in urlAttributes) {
          if (
            Object.hasOwn(urlAttributes, key) &&
            Object.hasOwn(node.properties, key)
          ) {
            const value = node.properties[key]
            const test = urlAttributes[key]
            if (test === null || test.includes(node.tagName)) {
              node.properties[key] = urlTransform(
                String(value || ''),
                key,
                node
              )
            }
          }
        }
      }

      if (node.type === 'element') {
        let remove = allowedElements
          ? !allowedElements.includes(node.tagName)
          : disallowedElements
          ? disallowedElements.includes(node.tagName)
          : false

        if (!remove && allowElement && typeof index === 'number') {
          remove = !allowElement(node, index, parent)
        }

        if (remove && parent && typeof index === 'number') {
          if (unwrapDisallowed && node.children) {
            parent.children.splice(index, 1, ...node.children)
          } else {
            parent.children.splice(index, 1)
          }

          return index
        }
      }
    }

    return () =>
      toJsxRuntime(hastTree, {
        Fragment,
        components,
        ignoreInvalidStyle: true,
        jsx,
        jsxs,
        passKeys: true,
        passNode: true,
        elementAttributeNameCase: 'html',
        stylePropertyNameCase: 'css',
      })
  },
}

function defaultUrlTransform(value) {
  // Same as:
  // <https://github.com/micromark/micromark/blob/929275e/packages/micromark-util-sanitize-uri/dev/index.js#L34>
  // But without the `encode` part.
  const colon = value.indexOf(':')
  const questionMark = value.indexOf('?')
  const numberSign = value.indexOf('#')
  const slash = value.indexOf('/')

  if (
    // If there is no protocol, it’s relative.
    colon < 0 ||
    // If the first colon is after a `?`, `#`, or `/`, it’s not a protocol.
    (slash > -1 && colon > slash) ||
    (questionMark > -1 && colon > questionMark) ||
    (numberSign > -1 && colon > numberSign) ||
    // It is a protocol, it should be allowed.
    safeProtocol.test(value.slice(0, colon))
  ) {
    return value
  }

  return ''
}
</script>
