Zend Framework 2 e Doctrine ORM
============

###Descrição

Este projeto contém um exemplo prático da utilização do [Zend Framework 2](http://framework.zend.com/manual/2.0/en/index.html) com o ORM [Doctrine](http://www.doctrine-project.org/). Trata-se da implementação de um [CRUD](http://en.wikipedia.org/wiki/Create,_read,_update_and_delete) simples de produtos, cuja finalidade é demonstrar a integração entre o ZF2 e Doctrine.

A configuração da máquina utilizada para realização deste tutorial foi:

* Ubuntu 13.04
* Apache 2.2.22
* MySQL 5.5.29
* PHP 5.4.6
* Git 1.7.10.4

## Preparação do ambiente

#### Obtendo o Zend Framework 2

Este tutorial assume que o local deste projeto será no diretório **/var/www**.

```
cd /var/www
git clone git@github.com:zendframework/ZendSkeletonApplication.git zf2-doctrine
```

## Instalando dependências

##### composer.json

Acrescentar as depencencias referentes ao doctrine no arquivo:

```
"doctrine/doctrine-orm-module": "~0.7.0",
"doctrine/migrations": "dev-master"
```

Desta forma, o arquivo, ficará da seguinte maneira:

```
{
    "name": "zendframework/skeleton-application",
    "description": "Skeleton Application for ZF2",
    "license": "BSD-3-Clause",
    "keywords": [
        "framework",
        "zf2"
    ],
    "homepage": "http://framework.zend.com/",
    "require": {
        "php": ">=5.3.3",
        "zendframework/zendframework": "2.*",
        "doctrine/doctrine-orm-module": "~0.7.0",
        "doctrine/migrations": "dev-master"
    }
}
```

Após efetuar as alterações no arquivo composer.json, basta executar o comando:

```
php composer.phar self-update && php composer.phar install
```

## VirtualHost

```
<VirtualHost *:80>
    ServerName zf2-doctrine.local
    DocumentRoot /var/www/zf2-doctrine/public

    SetEnv APPLICATION_ENV "development"
    SetEnv PROJECT_ROOT "/var/www/zf2-doctrine"

    <Directory "/var/www/zf2-doctrine/public">
        DirectoryIndex index.php
        AllowOverride All
        Order allow,deny
        Allow from all
    </Directory>

</VirtualHost>
```

## Hosts

```
echo "127.0.0.1 zf2-doctrine.local" >> /etc/hosts
```

## Database (script para geração da base de dados)

```
DROP DATABASE IF EXISTS zf2;
CREATE DATABASE zf2;
USE zf2;

DROP TABLE IF EXISTS `products`;
CREATE TABLE IF NOT EXISTS `products` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(100) NOT NULL,
  `description` text,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB  DEFAULT CHARSET=latin1 AUTO_INCREMENT=4 ;


INSERT INTO `products` (`id`, `name`, `description`) VALUES
(1, 'Achocolatado Nescau 2.0', 'NESCAU 2.0 é uma evolução do Nescau que todo mundo adora. Ele ganhou ainda mais vitaminas e um novo blend de Cacau surpreendente.'),
(2, 'Chocolate CHARGE', 'Combinação perfeita. Bombom de chocolate recheado com amendoim e caramelo.'),
(3, 'Chocolate Crunch', 'Chocolate ao leite NESTLÉ com flocos de arroz. O chocolate do barulho que todo mundo adora agora em versão 35g!');
```

## Configurando o projeto

#### config/autoload/local.php

```php
<?php
return array(
    'doctrine' => array(
        'connection' => array(
            'orm_default' => array(
                'driverClass' => 'Doctrine\DBAL\Driver\PDOMySql\Driver',
                'params' => array(
                    'host'     => 'localhost',
                    'port'     => '3306',
                    'user'     => 'root',
                    'password' => 'root',
                    'dbname'   => 'zf2',
                ),
            ),
        ),
    ),
);
```

#### Adicionando o módulo nas configurações da aplicação

```php
<?php
// config/application.config.php
return array(
    // This should be an array of module namespaces used in the application.
    'modules' => array(
        'Application',
        'DoctrineModule',       // Adicionar
        'DoctrineORMModule',    // Adicionar
        'Stock',                // Adicionar (módulo que iremos criar)
    ),
    .
    .
    .
```

## Criação do Módulo

Iremos criar um módulo do Zend Framework 2 para que possamos utilizar o Doctrine, portanto, dentro do diretório *zf2-doctrine/module* do projeto, devemos criar a seguinte estrutura de diretório:

```
Stock
  src
    Stock
      Controller
      Entity
  view
    stock
      product
```

Criando a estrutura de diretórios.

```
mkdir Stock
mkdir -p Stock/config
mkdir -p Stock/src/Stock/Controller
mkdir -p Stock/src/Stock/Entity
mkdir -p Stock/src/Stock/Form
mkdir -p Stock/view/stock/product
```

## Stock/Module.php
```php
<?php
namespace Stock;

class Module
{
    public function getAutoloaderConfig()
    {
        return array(
            'Zend\Loader\ClassMapAutoloader' => array(
                __DIR__ . '/autoload_classmap.php',
            ),
            'Zend\Loader\StandardAutoloader' => array(
                'namespaces' => array(
                    __NAMESPACE__ => __DIR__ . '/src/' . __NAMESPACE__,
                ),
            ),
        );
    }

    public function getConfig()
    {
        return include __DIR__ . '/config/module.config.php';
    }
    
}
```

## Stock/config/module.config.php

```php
<?php
namespace Stock;

return array(

    // Controllers in this module
    'controllers' => array(
        'invokables' => array(
            'Stock\Controller\Product' => 'Stock\Controller\ProductController'
        ),
    ),

    // Routes for this module
    'router' => array(
        'routes' => array(
            'product' => array(
                'type'    => 'segment',
                'options' => array(
                    'route'    => '/product[/:action][/:id]',
                    'constraints' => array(
                        'action' => '[a-zA-Z][a-zA-Z0-9_-]*',
                        'id'     => '[0-9]+',
                    ),
                    'defaults' => array(
                        'controller' => 'Stock\Controller\Product',
                        'action'     => 'index',
                    ),
                ),
            ),
        ),
    ),

    // View setup for this module
    'view_manager' => array(
        'template_path_stack' => array(
            'stock' => __DIR__ . '/../view',
        ),
    ),

    // Doctrine configuration
    'doctrine' => array(
        'driver' => array(
            __NAMESPACE__ . '_driver' => array(
                'class' => 'Doctrine\ORM\Mapping\Driver\AnnotationDriver',
                'cache' => 'array',
                'paths' => array(__DIR__ . '/../src/' . __NAMESPACE__ . '/Entity')
            ),
            'orm_default' => array(
                'drivers' => array(
                    __NAMESPACE__ . '\Entity' => __NAMESPACE__ . '_driver'
                ),
            ),
        ),
    ),

);
```

## Stock/autoload_classmap.php
```php
<?php
return array();
```

## Stock/src/Stock/Entity/Product.php
```php
<?php
/**
 * Tutorial of Zend Framework 2 and Doctrine
 *
 * This entity is a simple example how to use Doctrine in ZF2
 *
 * @author Thiago Pelizoni <thiago.pelizoni@gmail.com>
 */
namespace Stock\Entity;

use Doctrine\ORM\Mapping as ORM;
use Zend\InputFilter\InputFilter;
use Zend\InputFilter\Factory as InputFactory;
use Zend\InputFilter\InputFilterAwareInterface;
use Zend\InputFilter\InputFilterInterface; 

/**
 * Product
 *
 * @ORM\Entity
 * @ORM\Table(name="products")
 * @property int $id
 * @property string $name
 * @property string $description
 */
class Product implements InputFilterAwareInterface 
{
    /**
     * @var Zend\InputFilter\InputFilter
     */
    protected $inputFilter;

    /**
     * @ORM\Id
     * @ORM\Column(type="integer");
     * @ORM\GeneratedValue(strategy="AUTO")
     */
    protected $id;

    /**
     * @ORM\Column(type="string")
     */
    protected $name;

    /**
     * @ORM\Column(type="string")
     */
    protected $description;

    /**
     * Magic getter to expose protected properties.
     *
     * @param string $property
     * @return mixed
     */
    public function __get($property) 
    {
        return $this->$property;
    }

    /**
     * Magic setter to save protected properties.
     *
     * @param string $property
     * @param mixed $value
     */
    public function __set($property, $value) 
    {
        $this->$property = $value;
    }

    /**
     * Convert the object to an array.
     *
     * @return array
     */
    public function getArrayCopy() 
    {
        return get_object_vars($this);
    }

    /**
     * Populate from an array.
     *
     * @param array $data
     */
    public function populate(array $data = array()) 
    {
        foreach ($data as $property => $value) {
            if (! property_exists($this, $property)) {
                continue;
            }
            $this->$property = $value;
        }
    }

    public function setInputFilter(InputFilterInterface $inputFilter)
    {
        throw new \Exception("Not used!");
    }

    public function getInputFilter()
    {
        if (! $this->inputFilter) {
            $inputFilter = new InputFilter();

            $factory = new InputFactory();

            $inputFilter->add($factory->createInput(array(
                'name'       => 'id',
                'required'   => true,
                'filters' => array(
                    array('name'    => 'Int'),
                ),
            )));

            $inputFilter->add($factory->createInput(array(
                'name'     => 'name',
                'required' => true,
                'filters'  => array(
                    array('name' => 'StripTags'),
                    array('name' => 'StringTrim'),
                ),
                'validators' => array(
                    array(
                        'name'    => 'StringLength',
                        'options' => array(
                            'encoding' => 'UTF-8',
                            'min'      => 1,
                            'max'      => 100,
                        ),
                    ),
                ),
            )));

            $inputFilter->add($factory->createInput(array(
                'name'     => 'description',
                'required' => true,
                'filters'  => array(
                    array('name' => 'StripTags'),
                    array('name' => 'StringTrim'),
                ),
                'validators' => array(
                    array(
                        'name'    => 'StringLength',
                        'options' => array(
                            'encoding' => 'UTF-8',
                            'max'      => 1000,
                        ),
                    ),
                ),
            )));

            $this->inputFilter = $inputFilter;        
        }

        return $this->inputFilter;
    } 
}
```

## Stock/src/Stock/Form/ProductForm.php
```php
<?php
namespace Stock\Form;

use Zend\Form\Form;

class ProductForm extends Form
{
    public function __construct($name = null)
    {
        parent::__construct('Product');
        
        $this->setAttribute('method', 'post');
        
        $this->add(array(
            'name' => 'id',
            'attributes' => array(
                'type'  => 'hidden',
            ),
        ));
        
        $this->add(array(
            'name' => 'name',
            'attributes' => array(
                'type'  => 'text',
            ),
            'options' => array(
                'label' => 'Name',
            ),
        ));
        
        $this->add(array(
            'name' => 'description',
            'attributes' => array(
                'type'  => 'textarea',
            ),
            'options' => array(
                'label' => 'Description',
            ),
        ));
        
        $this->add(array(
            'name' => 'submit',
            'attributes' => array(
                'type'  => 'submit',
                'value' => 'Go',
                'id' => 'submitbutton',
            ),
        ));
    }
}
```

## Stock/src/Stock/Controller/ProductController.php
```php
<?php

namespace Stock\Controller;

use Zend\Mvc\Controller\AbstractActionController;
use Zend\View\Model\ViewModel; 
use Doctrine\ORM\EntityManager;
use Stock\Entity\Product;
use Stock\Form\ProductForm;

class ProductController extends AbstractActionController
{
    /**
     * @var Doctrine\ORM\EntityManager
     */
    protected $em;

    public function setEntityManager(EntityManager $em)
    {
        $this->em = $em;
    }
 
    /**
     * Return a EntityManager
     *
     * @return Doctrine\ORM\EntityManager
     */
    public function getEntityManager()
    {
        if ($this->em === null) {
            $this->em = $this->getServiceLocator()->get('Doctrine\ORM\EntityManager');
        }
        
        return $this->em;
    } 

    public function indexAction()
    {
        return new ViewModel(array(
            'products' => $this->getEntityManager()->getRepository('Stock\Entity\Product')->findAll() 
        ));
    }

    public function addAction()
    {
        $form = new ProductForm();
        $form->get('submit')->setAttribute('label', 'Add');

        $request = $this->getRequest();
        
        if ($request->isPost()) {
            $product = new Product();
            
            $form->setInputFilter($product->getInputFilter());
            $form->setData($request->getPost());
            
            if ($form->isValid()) { 
                $product->populate($form->getData()); 
                
                $this->getEntityManager()->persist($product);
                $this->getEntityManager()->flush();

                // Redirect to list of Stocks
                return $this->redirect()->toRoute('product'); 
            }
        }

        return array('form' => $form);
    }

    public function editAction()
    {
        $id = (int) $this->getEvent()->getRouteMatch()->getParam('id');
        
        if (!$id) {
            return $this->redirect()->toRoute('product', array('action'=>'add'));
        } 
        
        $Stock = $this->getEntityManager()->find('Stock\Entity\Product', $id);

        $form = new ProductForm();
        $form->setBindOnValidate(false);
        $form->bind($Stock);
        $form->get('submit')->setAttribute('label', 'Edit');
        
        $request = $this->getRequest();
        
        if ($request->isPost()) {
        
            $form->setData($request->getPost());
            
            if ($form->isValid()) {
                $form->bindValues();
                $this->getEntityManager()->flush();

                // Redirect to list of Stocks
                return $this->redirect()->toRoute('product');
            }
        }

        return array(
            'id' => $id,
            'form' => $form,
        );
    }

    public function deleteAction()
    {
        $id = (int)$this->getEvent()->getRouteMatch()->getParam('id');
        
        if (!$id) {
            return $this->redirect()->toRoute('product');
        }

        $request = $this->getRequest();
        
        if ($request->isPost()) {
            $del = $request->getPost('del', 'No');
            
            if ($del == 'Yes') {
                $id = (int) $request->getPost('id');
                $Stock = $this->getEntityManager()->find('Stock\Entity\Product', $id);
                
                if ($Stock) {
                    $this->getEntityManager()->remove($Stock);
                    $this->getEntityManager()->flush();
                }
            }

            return $this->redirect()->toRoute('product');
        }

        return array(
            'id' => $id,
            'product' => $this->getEntityManager()->find('Stock\Entity\Product', $id)
        );
    }
}
```
## Stock/view/stock/product/index.phtml
```php
<?php
$title = 'Products';
$this->headTitle($title);
?>
<h1><?php echo $this->escapeHtml($title); ?></h1>
<p>
    <a href="<?php echo $this->url('product', array('action'=>'add'));?>">Add new</a>
</p>

<table class="table">
<tr>
    <th>Name</th>
    <th>Description</th>
    <th>&nbsp;</th>
</tr>
<?php foreach ($products as $product) : ?>
<tr>
    <td><?php echo $this->escapeHtml($product->name);?></td>
    <td><?php echo $this->escapeHtml($product->description);?></td>
    <td>
        <a href="<?php echo $this->url('product',
            array('action'=>'edit', 'id' => $product->id));?>">Edit</a>
        <a href="<?php echo $this->url('product',
            array('action'=>'delete', 'id' => $product->id));?>">Delete</a>
    </td>
</tr>
<?php endforeach; ?>
</table>
```

## Stock/view/stock/product/add.phtml
```php
<?php
$title = 'Add new product';
$this->headTitle($title);
?>

<h1><?php echo $this->escapeHtml($title); ?></h1>

<?php
$form = $this->form;
$form->setAttribute('action', $this->url('product', array('action' => 'add')));
$form->prepare();

echo $this->form()->openTag($form);
echo $this->formHidden($form->get('id'));
echo $this->formRow($form->get('name'));
echo $this->formRow($form->get('description'));
echo $this->formSubmit($form->get('submit'));
echo $this->form()->closeTag();
```

## Stock/view/stock/product/edit.phtml
```php
<?php
$title = 'Edit product';
$this->headTitle($title);
?>
<h1><?php echo $this->escapeHtml($title); ?></h1>

<?php
$form = $this->form;
$form->setAttribute('action', $this->url(
    'product',
    array(
        'action' => 'edit',
        'id'     => $this->id,
    )
));
$form->prepare();

echo $this->form()->openTag($form);
echo $this->formHidden($form->get('id'));
echo $this->formRow($form->get('name'));
echo $this->formRow($form->get('description'));
echo $this->formSubmit($form->get('submit'));
echo $this->form()->closeTag();
```

## Stock/view/stock/product/delete.phtml
```php
<?php
$title = 'Delete product';
$this->headTitle($title);
?>
<h1><?php echo $this->escapeHtml($title); ?></h1>

<p>Are you sure that you want to delete '<?php echo $this->escapeHtml($product->name); ?>'</p>
<?php
$url = $this->url('product', array(
    'action' => 'delete',
    'id'     => $this->id,
));
?>
<form action="<?php echo $url; ?>" method="post">
    <div>
        <input type="hidden" name="id" value="<?php echo (int) $product->id; ?>" />
        <input type="submit" name="del" value="Yes" />
        <input type="submit" name="del" value="No" />
    </div>
</form>
```
