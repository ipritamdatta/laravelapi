
[ 
#to view data in json format you can you this google extension:
	https://chrome.google.com/webstore/detail/json-formatter/bcjindcccaagfpapjjmafapmmgkkhgoa?hl=bn

#pre-requisite: basic knowledge of laravel. 
]


LET'S BEGIN:

1) create a laravel project

2) created two model Product & Review with their respective factory,migration,seeder, controller using the following
line of code:

php artisan make:model Model/Product -a
php artisan make:model Model/Review -a

3) In Api.php file defined our api route for Product and Review:

Route::apiresource('/products', 'ProductController');
Route::group(['prefix'=>'products'],function(){
    Route::apiresource('/{product}/reviews','ReviewController'); // /products/11/reviews
    
});

4) you can see the routes in the command prompt using:

php artisan route:list

5) migration of product:


public function up()
    {
        Schema::create('products', function (Blueprint $table) {
            $table->bigIncrements('id');
            $table->string('name');
            $table->text('detail');
            $table->integer('price');
            $table->integer('stock');
            $table->integer('discount');
            $table->timestamps();
        });
    }


6) migration of review:

 public function up()
    {
        Schema::create('reviews', function (Blueprint $table) {
            $table->bigIncrements('id');
            $table->string('customer');
            $table->text('review');
            $table->integer('star');
            $table->bigInteger('product_id')->unsigned();
            $table->foreign('product_id')->references('id')->on('products')->onDelete('cascade');
            $table->timestamps();
        });
    }

7) php artisan migrate

#Now we will seed our database using factory and seeder command:

You can get faker related information from this github link: https://github.com/fzaninotto/Faker

8) In Product factory write: 
$factory->define(Product::class, function (Faker $faker) {
    return [
        'name' => $faker->word,
        'detail' => $faker->paragraph,
        'price' => $faker->numberBetween(100,1000),
        'stock' => $faker->randomDigit,
        'discount' => $faker->numberBetween(2,30)
    ];
});

9)In review factory write: 
$factory->define(Review::class, function (Faker $faker) {
    return [
        'product_id' => function(){
            return Product::all()->random();
        },
        'customer' => $faker->name,
        'review' => $faker->paragraph,
        'star'  => $faker->numberBetween(0,5)
    ];
});
10) In seeds-> databaseSeeder write:
public function run()
    {
        // $this->call(UsersTableSeeder::class);
        factory(App\Model\Product::class,50)->create();
        factory(App\Model\Review::class,300)->create();
    }

11) in command prompt write: 
	php artisan db:seed

#Creating Relationship

12) In product model write:
    public function reviews(){

        return $this->hasMany(Review::class);
    
    }

13) In review model write:
    public function product(){
        return $this->belongsTo(Product::class);
    }

14) To test the relationship, in command prompt write (Optional):
	php artisan tinker
	App/Model/Product::find(4) 
	App/Model/Product::find(4)->reviews




#Creating API Resource / Transformer:

15)Write this in the command prompt:
	 php artisan make:resource Product/ProductCollection

16)  Now to check all products; go to product controller and in index write:
	return Product::all();

17) Check this in the localhost's link: 
	http://localhost:8000/api/products

# Creating a new transformer:
18) in command prompt: 
	php artisan make:resource Product/ProductResource

19) In productResouce.php file write:
    public function toArray($request)
    {
        return [
            'name' => $this->name,
            'description' => $this->detail,
            'price' => $this->price,
            'stock' => $this->stock,
            'discount' => $this->discount
        ];
    }

20) In product controller:
use App\Http\Resources\Product\ProductResource;

public function show(Product $product)
    {
        return new ProductResource($product);
    }

21) open http://localhost:8000/api/products/4 and see how it transforms and showed data without
showin created_at updated_at.

#Transforming Products

22) To access particular product reviews as a link in json file:
 In ProductResource.php write:
return [
            'name' => $this->name,
            'description' => $this->detail,
            'price' => $this->price,
            'stock' => $this->stock,
            'discount' => $this->discount,
            'href' => [
                'reviews' => route('reviews.index', $this->id)
            ]
        ];

(write href part only)

23) To see the rating by making a logic here we are summing up all the star and showing
rating so write:
         return [
            'name' => $this->name,
            'description' => $this->detail,
            'price' => $this->price,
            'stock' => $this->stock == 0 ? 'Out of stock' : $this->stock,
            'discount' => $this->discount,
            'rating' => $this->reviews->count() > 0 ? round($this->reviews->sum('star')/$this->reviews->count(),2) : 'No rating yet',
            'href' => [
                'reviews' => route('reviews.index', $this->id)
            ]
        ];

24) To make a logic for total price we can write:

	return [
            'name' => $this->name,
            'description' => $this->detail,
            'price' => $this->price,
            'stock' => $this->stock == 0 ? 'Out of stock' : $this->stock,
            'discount' => $this->discount,
            'totalPrice' => round((1- ($this->discount/100)) * $this->price,2),
            'rating' => $this->reviews->count() > 0 ? round($this->reviews->sum('star')/$this->reviews->count(),2) : 'No rating yet',
            'href' => [
                'reviews' => route('reviews.index', $this->id)
            ]
        ];




#Product Collection Transforming


We want to show name, total price, rating, href only  in the index page and when user clicks the href it will show the full 
details of that product id.
So let's begin: 


25) Go to productCollection and instead of use Illuminate\Http\Resources\Json\ResourceCollection;
write use Illuminate\Http\Resources\Json\Resource;

and , instead of class ProductCollection extends ResourceCollection write 
class ProductCollection extends Resource

(because the Resource should match the ProductResource's Resource)

26) In productController index write:

    public function index()
    {
        return ProductCollection::collection(Product::all());
    }

and import this:
	use App\Http\Resources\Product\ProductCollection;


(Now, if you reload your localhost:8000/api/products you will not see the href of 
product resource as because we are returning from productcollection so do the following number 27)

27) In ProductCollection.php write:
        return [
            'name' => $this->name,
            'totalPrice' => round((1- ($this->discount/100)) * $this->price,2),
            'rating' => $this->reviews->count() > 0 ? round($this->reviews->sum('star')/$this->reviews->count(),2) : 'No rating yet',
            'discount' => $this->discount,
            'href' => route('products.show',$this->id)
        ];

28) In ProductResource.php write:

        return [
            'name' => $this->name,
            'description' => $this->detail,
            'price' => $this->price,
            'stock' => $this->stock == 0 ? 'Out of stock' : $this->stock,
            'discount' => $this->discount,
            'totalPrice' => round((1- ($this->discount/100)) * $this->price,2),
            'rating' => $this->reviews->count() > 0 ? round($this->reviews->sum('star')/$this->reviews->count(),2) : 'No rating yet',
            'href' => [
                'reviews' => route('reviews.index', $this->id)
            ]
        ];

NOW, In localhost:8000/api/products/ you will see that each data has some info
and a link. When you click on that link you will be able to see the full details of that particular
data.


Till now we have products to product details, to we will do product details to review details

#Get Review API Resource: 
Now, we will show the review for that particular product and obviously that has to be 
transformed.


29) Open command prompt and write:
	php artisan make:resource ReviewResource

30) In reviewController write:

    use App\Model\Review;
    public function index(Product $product)
    {
        return $product->reviews; //here the reviews came from product.php relationship
    }

(See the link http://localhost:8000/api/products/25/reviews)

31) Now we want to show only customer, review and star in the particular id's review page. So,

go to reviewResource and write:

        return [
            'customer' => $this->customer,
            'body' => $this->review,
            'star' => $this->star
        ];

and for this we have to declare that in the reviewController as well. So,

32) Go to reviewController and write:

import this: 
	 use App\Http\Resources\ReviewResource;
    
    public function index(Product $product)
    {
        return ReviewResource::collection($product->reviews);
    }

(You can see the review of that particular product:        "http://localhost:8000/api/products/25/reviews")


33) There can be so many products so let's do the pagination: 

    In ProductController write: 

    public function index()
    {
        // return ProductResource::collection(Product::all());
        return ProductCollection::collection(Product::paginate(20));
    }



#get introduced to postman (Optional) but let's do this for better experience:

install postman app: https://www.postman.com/

 open postman

and in the get method paste the link: http://localhost:8000/api/products and click send.
This will show exactly the same as our browser.

Create a collection in postman naming EAPI and inside that create a folder called Products.


and inside Products folder we have http://localhost:8000/api/products/ this link in the 
get request. so, 

 click save and name the request as Product All. scroll below and
select the Products folder. and click save.

again, in the GET request write http://localhost:8000/api/products/4 and again
click save as and name it like Product Show. select EAPI's Product and click save.


In same way add a Review folder inside the EAPI collection and inside Review folder's 
Get Request paste http://localhost:8000/api/products/4/reviews and save as Review All.


#Configure passport package


Laravel already makes it easy to perform authentication via traditional login forms, 
but what about APIs? APIs typically use tokens to authenticate users and do not maintain session state between requests. 
Laravel makes API authentication a breeze using Laravel Passport, which provides a full OAuth2 server implementation 
for your Laravel application in a matter of minutes. 

34) Now in the command prompt install laravel passport: 
	composer require laravel/passport

35) The Passport service provider registers its own database migration directory with the framework, so you should migrate your database after installing the package.
 The Passport migrations will create the tables your application needs to store clients and access tokens:

After installation run this,	php artisan migrate

(See the tables in eapi database of your local server)

36) run this command: php artisan passport:install

This command will create the encryption keys needed to generate secure access tokens.
You can see the tokens in the oauth_clients file of your local server.

37) In User.php write: 

class User extends Authenticatable
{
    use HasApiTokens, Notifiable;
}

38) In auth service provider write this: 
use Laravel\Passport\Passport;

public function boot()
    {
        $this->registerPolicies();

        Passport::routes();
    }


39) In auth.php write:
'guards' => [
        'web' => [
            'driver' => 'session',
            'provider' => 'users',
        ],

        'api' => [
            'driver' => 'passport',
            'provider' => 'users',
            'hash' => false,
        ],
    ],

here we have written passport instead of token

Open postman and create a folder named OAUTH inside EAPI
and instead of GET request select POST request and paste this: http://localhost:8000/oauth/token/
and in the Headers section write Accept in KEY and write application/json in VALUE.
again, in the Headers section write Content-Type in KEY and write application/json in VALUE.

40) In command prompt,
 	composer require laravel/ui
	php artisan ui vue --auth

then, npm install && npm run dev (if your register page can't find the design css)

41) register as a new user in localhost. 

Now in postman write go to body section of OAUTH folder and select Raw option. write:

{
	"grant_type" : "password",
	"client_id": "2",
	"client_secret" : "XlSQdAUeM315AFlFayrk35gWHaU1TO3zWtrD2Sdi",
	"username" : "pritamdattashuvo@gmail.com",
	"password" : "mypassword"
}

and click send.

Save this inside OAUTH folder naming Get Token.

now copy the full access_token which is like 
	eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiJ9.eyJh....

and to check if user is authenticated or not open another tab in postman and 
select get request and paste it: http://localhost:8000/api/user
and in header section write Accept, Content-Type, Authorization (unselect checkmark of authorization),
and in the Value section write application/json. and random name and space paste the token
Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiJ9.eyJh....

{Remember to write Bearer otherwise you will see unauthenticated all the time}

This will unauthenticated as it is not registered


Now, select the checkmark and click send, you will see who is authenticated user.



Now, click right side setting button and create an envirnment named EAPI
and select it.

write from 

# Creating new product

# Updating product

# Delete product

# Handle Exceptions
  Error, Responses and Exceptions

# Custom Exceptions

# Authorise Product Update


