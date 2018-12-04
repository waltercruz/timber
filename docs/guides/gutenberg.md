---
title: "Gutenberg"
menu:
  main:
    parent: "guides"
---

## Using Gutenberg with Timber

Timber works with Gutenberg out of the box. If you use `{{ post.content }}`, Timber will render all the Gutenberg blocks.

If you want to go further and work with existing blocks or create your own blocks, Timber can help you.

## Render callbacks

A render callback allows you to intercept the rendering of each block in the frontend. In the following example, we will register a render callback for the `core/image` block. Every time the block is rendered, it will be piped through our callback. This way, we can intercept the output of the block’s content, and add our own markup.

```php
register_block_type( 'core/image', array(
    'render_callback' => 'core_image_render_callback'
) );

/**
 * Renders the core/image block.
 *
 * @param array  $attributes The block props.
 * @param string $content    The block content.
 * @return string
 */
function core_image_render_callback( $attributes, $content ) {
    return Timber::compile( 'block/image.twig', array(
        'attributes' => $attributes,
        'content'    => $content,
    ) );
}
```

The `$content` variable will hold the markup for the image, that was saved in the [`save()`](https://wordpress.org/gutenberg/handbook/designers-developers/developers/block-api/block-edit-save/#save) function of the block. This means that using this technique is mostly limited to **wrapping existing blocks** with your own markup.

### Attributes

For rendering callbacks it’s important to understand what the `$attributes` parameter is. In Gutenberg, you can define sources for all attributes that you define. As explained in [the documentation](https://wordpress.org/gutenberg/handbook/designers-developers/developers/block-api/block-attributes/#common-sources):

> If no attribute source is specified, the attribute will be saved to (and read from) the block’s comment delimiter.

Consider the following markup:

```html
<!-- wp:my-plugin/image {"attachmentId":68} /-->
<!-- /wp:my-plugin/image-text -->
```

This is a block that was defined in the following way:

```php
register_block_type( 'my-plugin/image', array(
    'attributes' => [
        'attachmentId' => [
            'type'    => 'integer',
            'default' => '',
        ],
    ],
    'render_callback' => 'my_plugin_image_render_callback',
) );
```

The block has an attribute `attachmentId` that will be saved in the HTML markup. It also has an empty [`save()`](https://wordpress.org/gutenberg/handbook/designers-developers/developers/block-api/block-edit-save/#save) function. That’s why it has no content yet. By using the render callback function, we can set the content dynamically:

```php
/**
 * Renders the block.
 *
 * @param array  $attributes The block props.
 * @param string $content    The block content.
 * @return string
 */
function my_plugin_image_render_callback( $attributes, $content = '' ) {
    if ( empty( $attributes['attachmentId'] ) ) {
        return $content;
    }

    $image = new Timber\Image( $attributes['attachmentId'] );

    return Timber::compile( 'block/image.twig', [
        'image' => $image,
    ] );
}
```

In your Twig template, you then have complete freedom over your markup.

**block/image.twig**

```twig
{% if image.caption %}
    <figure>
        <img src="{{ image.src('large') }}" alt="{{ image.alt }}">
        <figcaption>{{ image.caption }}</figcaption>
    </figure>
{% else %}
    <img src="{{ image.src('large')) }}" alt="{{ image.alt }}">
{% endif %}
```

With this technique you need to consider that your attributes need to be defined in PHP and not in JavaScript.

## ACF Blocks

ACF Blocks are an alternative way to create content blocks without advanced JavaScript knowledge. If you want to learn more about them, read the article on [advancedcustomfields.com](https://www.advancedcustomfields.com/blog/acf-5-8-introducing-acf-blocks-for-gutenberg/).

### How to use ACF Blocks with Timber

Before you can start using ACF Blocks, you must install the Advanced Custom Fields 5.8.0-beta version or later.

To create a content block, you first have to register it in **functions.php** or in a separate plugin:

```php
add_action( 'acf/init', 'my_acf_init' );

function my_acf_init() {
    // Check function exists.
    if ( function_exists( 'acf_register_block' ) ) {
        // Register a new block.
        acf_register_block(array(
            'name'				=> 'example_block',
            'title'				=> __( 'Example Block', 'your-text-domain' ),
            'description'		=> __( 'A custom example block.', 'your-text-domain' ),
            'render_callback'	=> 'my_acf_block_render_callback',
            'category'			=> 'formatting',
            'icon'				=> 'admin-comments',
            'keywords'		    => array( 'example' ),
        ) );
    }
}
```

Next, you you have to create your `render_callback()` function:

```php
function my_acf_block_render_callback( $block ) {
    $context = Timber::get_context();
    
    // Store block values.
    $context['block'] = $block;

    // Store field values.
    $context['fields'] = get_fields(); 

    // Render the block.
    Timber::render( 'block/example-block.twig', $context );
}
```

You create an extra array called `$context` with two values:
- **block** - with all data like block title, alignment etc
- **fields** - all custom fields - also all the fields created in **ACF**

Finally, you can create the template **block/example-block.twig**:

```twig
{#
/**
 * Block Name: Example block
 *
 * This is the template that displays the example block.
 */
#}
<div id="example-{{ block.id }}" class="wrapper">
    <h1>{{ fields.title }}</h1>
    <p>{{ fields.description }}</p>
</div>
<style type="text/css">
    #testimonial-{{ block.id }} {
        background: {{ fields.background_color }};
        color: {{ fields.text_color }};
    }
</style>
```

### Using repeaters

```
{% for field in fields.repeater %}
    Title: {{ field.title }} <br/>
    Url: {{ field.url }}
{% endfor %}
```

### Using groups

```
Title: {{ fields.group.title }} <br/>
Url: {{ fields.group.url }}
```

## Styling

If you would like to use an external stylesheet both inside of the block editor and the frontend you should add:

```php
function my_acf_block_editor_style() {
    wp_enqueue_style(
        'logo_grid_css',
        get_template_directory_uri() .'/assets/example-block.css'
    );
}

add_action( 'enqueue_block_assets', 'my_acf_block_editor_style' );
```

For more details about enqueueing assets read the [Gutenberg Handbook](https://wordpress.org/gutenberg/handbook/blocks/applying-styles-with-stylesheets/#enqueueing-editor-only-block-assets).

## Dynamic blocks

[Dynamic blocks](https://wordpress.org/gutenberg/handbook/designers-developers/developers/tutorials/block-tutorial/creating-dynamic-blocks/) work well with Timber, too.

While server side rendering is not really suggested by the Gutenberg developers, it’s an approach that allows you to re-use your Timber templates in both the editor and the frontend.

You can either render the whole block, or only parts of a block. Let’s have a look a the the [Latest Posts Block example](https://wordpress.org/gutenberg/handbook/designers-developers/developers/tutorials/block-tutorial/creating-dynamic-blocks/) from the documentation and how this would look like in Timber.

### A latests posts block

Instead of rendering the list of posts in the edit mode through JavaScript, you can use the [ServerSideRenderer](https://wordpress.org/gutenberg/handbook/designers-developers/developers/components/server-side-render/) component to do that for you.

```js
const { registerBlockType } = wp.blocks;
const { ServerSideRender } = wp.components;
const { Fragment } = wp.element;
const { __ } = wp.i18n;

registerBlockType('my-plugin/latest-posts', {
  title: __('The latest posts', 'my-plugin'),
  category: 'common',
  icon: 'excerpt-view',

  supports: {
    html: false
  },

  edit: ({ attributes, className }) => {
    return (
      <Fragment>
        <div className={className}>
          <ServerSideRender
            block="my-plugin/latest-posts"
            attributes={attributes}
          />
        </div>
      </Fragment>
    );
  },

  // Rendering in PHP
  save: () => null
});
```

Now, you would register a callback for the block in PHP. In the callback function, you can run a custom query and then use `Timber::compile()` to return the contents of that template.

```php
register_block_type( 'my-plugin/latest-posts', array(
    'render_callback' => 'my_plugin_render_block_latest_post',
) );

/**
 * Renders the block.
 *
 * @param array  $attributes The block props.
 * @param string $content    The block content.
 * @return string
 */
function my_plugin_render_block_latest_post( $attributes, $content = '' ) {
    $posts = new Timber\PostQuery( array(
        'post_type'      => 'post',
        'post_status'    => 'publish',
        'posts_per_page' => 3,
    ) );

    return Timber::compile( 'block/latest-posts.twig', array(
        'posts' => $posts,
    ) );
}
```

In this example, no attributes are used for brevity. But of course you extend the block with custom attributes. For example, if you wanted to automate the `posts_per_page` property in the query, you could add it as an [Attribute](https://wordpress.org/gutenberg/handbook/designers-developers/developers/block-api/block-attributes/) in the block that you pass into the query.

**block/latest-posts.twig**

```twig
<h2 class="h2">{{ __('Latest posts', 'my-plugin') }}</h2>

<ul class="latest-posts-items">
    {% for post in posts %}
        <li class="latest-posts-item">
            <a class="latest-posts-link" href="{{ post.link }}">
                <h3 class="heading-3">{{ post.title }}</h3>
            </a>
        </li>
    {% endfor %}
</ul>

<a href="{{ fn('home_url') }}">{{ __('All posts', 'my-plugin') }}</a>
```
