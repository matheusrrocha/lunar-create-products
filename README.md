# Lunar - Create products programmatically

Here's an exemple of how to create a product programmatically. It will consider a product with color and size variant (very common and covers a lot).

**I'm assuming you already have a working project with `laravel` and `lunar` instaled**

## 1. Create the color variant

```php
use Lunar\Models\ProductOption;

$colorVariant = ProductOption::create([
    'name' => [
        'en' => 'Color',
    ],
    'label' => [
        'en' => 'Color',
    ],
    'handle' => 'color',
    'shared' => true
]);

$colorVariant->values()->createMany([
    [
        'name' => [
            'en' => 'Black',
        ],
    ],
    [
        'name' => [
            'en' => 'White',
        ],
    ],
    [
        'name' => [
            'en' => 'Pale Pink',
        ],
    ],
    [
        'name' => [
            'en' => 'Mid Blue',
        ],
    ],
]);
```

## 2. Create the size variant

```php
use Lunar\Models\ProductOption;

$sizeVariant = ProductOption::create([
    'name' => [
        'en' => 'Size',
    ],
    'label' => [
        'en' => 'Size',
    ],
    'handle' => 'size',
    'shared' => true
]);

$sizeVariant->values()->createMany([
    [
        'name' => [
            'en' => 'XS',
        ],
    ],
    [
        'name' => [
            'en' => 'S',
        ],
    ],
    [
        'name' => [
            'en' => 'M',
        ],
    ],
    [
        'name' => [
            'en' => 'L',
        ],
    ],
    [
        'name' => [
            'en' => 'XL',
        ],
    ],
]);
```

## 3. Product type

```php
use Lunar\Models\ProductType;

$productType = ProductType::create([
    'name' => 'T-Shirt',
]);
```

## 4. Create the product

```php
use Lunar\FieldTypes\TranslatedText;
use Lunar\FieldTypes\Text;

$brand = Brand::first(); //dont forget to create a Lunar\Models\Brand
$sku = 'SKU-1234';

$product = Product::create([
    'product_type_id' => $productType->id,
    'status' => 'published',
    'brand_id' => $brand->id,
    'sku' => $sku,
    'attribute_data' => [
        'name' => new TranslatedText(collect([
            'en' => new Text('Name'), //assuming your store have the EN as default language
        ])),
        'description' => new Text('Description'),
    ],
]);
```

## 4. Add the variants to the product

There's probably a better way to create this records but i didnt had the time to look and this works, so...

```php
//Add color
DB::table('lunar_product_product_option')->insert([
    'product_id' => $product->id,
    'product_option_id' => $colorVariant->id,
    'position' => 1
]);

//Add size
DB::table('lunar_product_product_option')->insert([
    'product_id' => $product->id,
    'product_option_id' => $sizeVariant->id,
    'position' => 2
]);
```

## 5. Add the combinations of colors X sizes

Here's the trick part

```php
use Lunar\Models\ProductVariant;
use Lunar\Models\TaxClass;
use Lunar\Models\Currency;

$colors = $colorVariant->values;
$sizes = $sizeVariant->values;

$colors->values()->each(function ($color) use ($sizes, $product, $sku) {
    $sizes->values()->each(function ($size) use ($color, $product, $sku) {
        $this->createProductVariant($product, $color, $size, $sku);
    });
});

protected function createProductVariant($product, $color, $size, $sku)
{
    //create the variant
    $variant = ProductVariant::create([
        'product_id' => $product->id,
        'tax_class_id' => TaxClass::first()->id, //also make sure you already have a use Lunar\Models\TaxClass create
        'sku' => "$sku-{$color->translate('name')}-{$size->translate('name')}", //here you can create any sku for the variant, for exemple a combination the the original sku + color and size 
    ]);

    //add a price to the variant
    $variant->prices()->create([
        'price' => (int) bcmul(fake()->numerify('###'), 100),
        'currency_id' => Currency::first()->id, //check if you have at least one currency
    ]);

    //sync colors X sizes. This is crucial to make sure all relations are created properly
    $variant->values()->sync([
        $color->id,
        $size->id,
    ]);
}
```