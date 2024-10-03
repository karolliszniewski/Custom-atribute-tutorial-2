### Magento 2 add custom attribute to configurable product options

1. First, let's create the module structure:

```bash
Acme_ProductOptionVideo/
│
├── registration.php
├── etc/
│   ├── module.xml
│   ├── db_schema.xml
│   └── adminhtml/
│       └── di.xml
│
├── Model/
│   └── Product/
│       └── Attribute/
│           └── Backend/
│               └── VideoLink.php
│
├── Setup/
│   └── Patch/
│       └── Data/
│           └── AddVideoLinkAttribute.php
│
└── view/
    └── adminhtml/
        └── ui_component/
            └── product_form.xml
```

2. Create `registration.php`: This file registers the module with Magento's module system. It tells Magento that our module exists and where it's located in the file system.

```php
<?php
use Magento\Framework\Component\ComponentRegistrar;

ComponentRegistrar::register(
    ComponentRegistrar::MODULE,
    'Acme_ProductOptionVideo',
    __DIR__
);
```

3. Create `etc/module.xml`: 
This file declares the module to Magento, specifying its name and version. It also establishes that our module depends on the Magento_Catalog module, ensuring it loads after the catalog module.

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="urn:magento:framework:Module/etc/module.xsd">
    <module name="Acme_ProductOptionVideo" setup_version="1.0.0">
        <sequence>
            <module name="Magento_Catalog"/>
        </sequence>
    </module>
</config>
```

4. Create `etc/db_schema.xml`:

```xml
<?xml version="1.0"?>
<schema xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="urn:magento:framework:Setup/Declaration/Schema/etc/schema.xsd">
    <table name="catalog_product_entity_varchar" resource="default" engine="innodb">
        <column xsi:type="varchar" name="video_link" nullable="true" length="255" comment="Video Link"/>
    </table>
</schema>
```

5. Create `Model/Product/Attribute/Backend/VideoLink.php`: This is a backend model for our custom attribute. It handles how the attribute data is saved to and loaded from the database. Specifically:

The afterLoad method decodes the JSON-stored video links when a product is loaded.
The beforeSave method encodes the video links as JSON before saving to the database.

```xml
<?php
namespace Acme\ProductOptionVideo\Model\Product\Attribute\Backend;

use Magento\Eav\Model\Entity\Attribute\Backend\AbstractBackend;
use Magento\Framework\DataObject;

class VideoLink extends AbstractBackend
{
    public function afterLoad($object)
    {
        $attributeCode = $this->getAttribute()->getAttributeCode();
        $value = $object->getData($attributeCode);
        if (!is_array($value)) {
            $value = $value ? json_decode($value, true) : [];
        }
        $object->setData($attributeCode, $value);
        return $this;
    }

    public function beforeSave($object)
    {
        $attributeCode = $this->getAttribute()->getAttributeCode();
        $value = $object->getData($attributeCode);
        if (is_array($value)) {
            $object->setData($attributeCode, json_encode($value));
        }
        return $this;
    }
}
```

6. Create `Setup/Patch/Data/AddVideoLinkAttribute.php`: 
This data patch creates the new video_link attribute for products. It sets up the attribute with specific properties like its type, label, and backend model. This patch runs when the module is installed or upgraded.

```php
<?php
namespace Acme\ProductOptionVideo\Setup\Patch\Data;

use Magento\Eav\Setup\EavSetup;
use Magento\Eav\Setup\EavSetupFactory;
use Magento\Framework\Setup\ModuleDataSetupInterface;
use Magento\Framework\Setup\Patch\DataPatchInterface;

class AddVideoLinkAttribute implements DataPatchInterface
{
    private $moduleDataSetup;
    private $eavSetupFactory;

    public function __construct(
        ModuleDataSetupInterface $moduleDataSetup,
        EavSetupFactory $eavSetupFactory
    ) {
        $this->moduleDataSetup = $moduleDataSetup;
        $this->eavSetupFactory = $eavSetupFactory;
    }

    public function apply()
    {
        /** @var EavSetup $eavSetup */
        $eavSetup = $this->eavSetupFactory->create(['setup' => $this->moduleDataSetup]);

        $eavSetup->addAttribute(
            \Magento\Catalog\Model\Product::ENTITY,
            'video_link',
            [
                'type' => 'varchar',
                'label' => 'Video Link',
                'input' => 'text',
                'required' => false,
                'sort_order' => 50,
                'global' => \Magento\Eav\Model\Entity\Attribute\ScopedAttributeInterface::SCOPE_STORE,
                'used_in_product_listing' => true,
                'backend' => \Acme\ProductOptionVideo\Model\Product\Attribute\Backend\VideoLink::class,
                'visible_on_front' => false
            ]
        );
    }

    public static function getDependencies()
    {
        return [];
    }

    public function getAliases()
    {
        return [];
    }
}
```

7. Create `etc/adminhtml/di.xml`: This file sets up a plugin that modifies the product editing form in the admin panel. It tells Magento to use our custom Eav plugin class when rendering the product form.

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">
    <type name="Magento\Catalog\Ui\DataProvider\Product\Form\Modifier\Eav">
        <plugin name="acme_product_option_video_eav_modifier" type="Acme\ProductOptionVideo\Plugin\Catalog\Ui\DataProvider\Product\Form\Modifier\Eav" />
    </type>
</config>
```

8. Create a new file `Plugin/Catalog/Ui/DataProvider/Product/Form/Modifier/Eav.php`: This is the core of our functionality. It modifies the product editing form in the admin panel. Here's what it does:

The afterModifyMeta method adds a new input field for each color option of the product. Each field is labeled "Video Link for [Color]" and is tied to a specific color option.
The afterModifyData method handles loading the saved video link data into these fields when editing a product.

```php
<?php
namespace Acme\ProductOptionVideo\Plugin\Catalog\Ui\DataProvider\Product\Form\Modifier;

use Magento\Catalog\Ui\DataProvider\Product\Form\Modifier\Eav as Subject;
use Magento\Ui\Component\Form\Element\Input;
use Magento\Ui\Component\Form\Field;

class Eav
{
    public function afterModifyMeta(Subject $subject, array $meta)
    {
        $attributes = $subject->getAttributes();
        foreach ($attributes as $attribute) {
            if ($attribute->getAttributeCode() == 'color') {
                $attributeCode = $attribute->getAttributeCode();
                $groupCode = $subject->getGroupCodeByAttribute($attribute);
                $groupName = 'container_' . $attributeCode;

                if (isset($meta[$groupCode]['children'][$groupName]['children'][$attributeCode]['arguments']['data']['config']['usedForConfigurableAttributes'])) {
                    $options = $attribute->getOptions();
                    array_shift($options); // Remove the first option which is usually empty

                    foreach ($options as $option) {
                        $optionId = $option->getValue();
                        $meta[$groupCode]['children'][$groupName]['children'][$attributeCode . '_' . $optionId . '_video_link'] = [
                            'arguments' => [
                                'data' => [
                                    'config' => [
                                        'label' => __('Video Link for %1', $option->getLabel()),
                                        'componentType' => Field::NAME,
                                        'formElement' => Input::NAME,
                                        'dataScope' => 'video_link.' . $optionId,
                                        'dataType' => 'text',
                                        'sortOrder' => 50,
                                    ],
                                ],
                            ],
                        ];
                    }
                }
            }
        }

        return $meta;
    }

    public function afterModifyData(Subject $subject, array $data)
    {
        foreach ($data as &$productData) {
            if (isset($productData['product']['video_link'])) {
                $videoLinks = json_decode($productData['product']['video_link'], true);
                if (is_array($videoLinks)) {
                    $productData['product']['video_link'] = $videoLinks;
                }
            }
        }
        return $data;
    }
}
```


This module will add a "Video Link" field for each color option of your configurable products in the admin panel. The video links will be stored as a JSON string in the video_link attribute of the product.
To display these video links on the frontend, you'll need to create a custom template that reads the video_link attribute, decodes the JSON, and displays the appropriate link based on the selected color.
Remember to adjust the attribute code ('color' in this example) if your color attribute has a different code in your Magento installation.
This solution provides a flexible way to add video links to specific product options. You can easily extend this to work with other attributes besides color by modifying the `Eav.php` plugin.







Namespace and Use Statements:
```php
namespace Acme\ProductOptionVideo\Plugin\Catalog\Ui\DataProvider\Product\Form\Modifier;
use Magento\Catalog\Ui\DataProvider\Product\Form\Modifier\Eav as Subject;
use Magento\Ui\Component\Form\Element\Input;
use Magento\Ui\Component\Form\Field;
```

This section defines the namespace of the class and imports necessary Magento classes. The class is extending Magento's core EAV (Entity-Attribute-Value) form modifier.

```php
Class Definition:

phpCopyclass Eav
{
    // Methods will be here
}
```

This defines our plugin class named Eav.

The afterModifyMeta Method:

```php
public function afterModifyMeta(Subject $subject, array $meta)
{
    // Method body
}
```

This is a plugin method that runs after Magento's core modifyMeta method. It allows us to modify the form metadata.
Let's break down the method body:
a. Getting attributes:

```php
$attributes = $subject->getAttributes();
```

This retrieves all attributes of the product.
b. Looping through attributes:
```php
foreach ($attributes as $attribute) {
    if ($attribute->getAttributeCode() == 'color') {
        // Process color attribute
    }
}
```

This loops through all attributes, looking specifically for the 'color' attribute.
c. Setting up attribute metadata:

```php
$attributeCode = $attribute->getAttributeCode();
$groupCode = $subject->getGroupCodeByAttribute($attribute);
$groupName = 'container_' . $attributeCode;
```

This prepares variables needed to locate and modify the correct part of the form.
d. Checking if the attribute is used for configurable products:

```php
if (isset($meta[$groupCode]['children'][$groupName]['children'][$attributeCode]['arguments']['data']['config']['usedForConfigurableAttributes'])) {
    // Process configurable attribute options
}
```
This ensures we only modify the form for configurable attributes.
e. Processing color options:
```php
$options = $attribute->getOptions();
array_shift($options); // Remove the first option which is usually empty

foreach ($options as $option) {
    // Add video link field for each color option
}
```
This gets all color options and adds a video link field for each one.
f. Adding video link field:
```php
$optionId = $option->getValue();
$meta[$groupCode]['children'][$groupName]['children'][$attributeCode . '_' . $optionId . '_video_link'] = [
    'arguments' => [
        'data' => [
            'config' => [
                'label' => __('Video Link for %1', $option->getLabel()),
                'componentType' => Field::NAME,
                'formElement' => Input::NAME,
                'dataScope' => 'video_link.' . $optionId,
                'dataType' => 'text',
                'sortOrder' => 50,
            ],
        ],
    ],
];
```
This creates a new input field for each color option, labeled "Video Link for [Color]".
The afterModifyData Method:

```php
public function afterModifyData(Subject $subject, array $data)
{
    foreach ($data as &$productData) {
        if (isset($productData['product']['video_link'])) {
            $videoLinks = json_decode($productData['product']['video_link'], true);
            if (is_array($videoLinks)) {
                $productData['product']['video_link'] = $videoLinks;
            }
        }
    }
    return $data;
}
```
This method runs after Magento's core modifyData method. It decodes the JSON-stored video links into an array when loading product data.
Tutorial: Adding Custom Fields to Product Edit Form in Magento 2
In this tutorial, we'll learn how to add custom fields to the product edit form in Magento 2 admin panel. We'll focus on adding video link fields for each color option of a configurable product.
Step 1: Create the Plugin Class
Create a new PHP file named Eav.php in your module's Plugin/Catalog/Ui/DataProvider/Product/Form/Modifier/ directory.
Step 2: Define the Class Structure
```php
namespace Acme\ProductOptionVideo\Plugin\Catalog\Ui\DataProvider\Product\Form\Modifier;

use Magento\Catalog\Ui\DataProvider\Product\Form\Modifier\Eav as Subject;
use Magento\Ui\Component\Form\Element\Input;
use Magento\Ui\Component\Form\Field;

class Eav
{
    public function afterModifyMeta(Subject $subject, array $meta)
    {
        // We'll add code here
    }

    public function afterModifyData(Subject $subject, array $data)
    {
        // We'll add code here
    }
}
```
Step 3: Implement the afterModifyMeta Method
This method will add our custom fields to the form:
```php
public function afterModifyMeta(Subject $subject, array $meta)
{
    $attributes = $subject->getAttributes();
    foreach ($attributes as $attribute) {
        if ($attribute->getAttributeCode() == 'color') {
            $attributeCode = $attribute->getAttributeCode();
            $groupCode = $subject->getGroupCodeByAttribute($attribute);
            $groupName = 'container_' . $attributeCode;

            if (isset($meta[$groupCode]['children'][$groupName]['children'][$attributeCode]['arguments']['data']['config']['usedForConfigurableAttributes'])) {
                $options = $attribute->getOptions();
                array_shift($options); // Remove the first option which is usually empty

                foreach ($options as $option) {
                    $optionId = $option->getValue();
                    $meta[$groupCode]['children'][$groupName]['children'][$attributeCode . '_' . $optionId . '_video_link'] = [
                        'arguments' => [
                            'data' => [
                                'config' => [
                                    'label' => __('Video Link for %1', $option->getLabel()),
                                    'componentType' => Field::NAME,
                                    'formElement' => Input::NAME,
                                    'dataScope' => 'video_link.' . $optionId,
                                    'dataType' => 'text',
                                    'sortOrder' => 50,
                                ],
                            ],
                        ],
                    ];
                }
            }
        }
    }

    return $meta;
}
```
Step 4: Implement the afterModifyData Method
This method will handle loading the saved video link data:
```php
public function afterModifyData(Subject $subject, array $data)
{
    foreach ($data as &$productData) {
        if (isset($productData['product']['video_link'])) {
            $videoLinks = json_decode($productData['product']['video_link'], true);
            if (is_array($videoLinks)) {
                $productData['product']['video_link'] = $videoLinks;
            }
        }
    }
    return $data;
}
```
That's it! This plugin will now add a "Video Link" field for each color option of configurable products in the admin product edit form. The video links will be stored as a JSON string in the database and loaded as an array when editing the product.
Remember to clear your cache and compile after adding this file to see the changes in the admin panel.




