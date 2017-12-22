For many web applications, database can be a serious bottleneck. In our photo sharing demo, usually the number of view image request is much greater than the number of upload requests. It is very possible that for many view requests, the most recent N images are actually the same. However, we are connecting to the database to fetch records for the most recent N images for each and every view requests. It would be reasonable to update the images we show on the index page in an incremental way, for example, every 1 or 2 minutes.

In this level, we will add a cache layer between the web servers and the database. When we fetch records for the most recent N images, we cache it somewhere. When there is a new view request coming in, we no longer connect to the database, but return the cached result to the user. When there is a new image upload, we update the cache. This way the cache version is always accurate.

The demo code has support for database caching through ElastiCache, using the same ElastiCache instance for session sharing. This caching behavior is not enable by default. You can edit config.php on all web servers with details regarding the cache server:

```
$enable_cache = true;
$cache_server  = "dns-name-of-your-elasticache-instance";
```

Refresh the demo application in your browser, you will see that the “Fetching N records from the database.” message is now gone, indicating that the information you are seeing is obtained from ElastiCache. When you upload a new image, you will see this message again, indicating the cache is being updated.

The following code is responsible of handling this cache logic:

```php
// Get the most recent N images
if ($enable_cache)
{
	// Attemp to get the cached records for the front page
	$mem = open_memcache_connection($cache_server);
	$images = $mem->get("front_page");
	if (!$images)
	{
		// If there is no such cached record, get it from the database
		$images = retrieve_recent_uploads($db, 10);
		// Then put the record into cache
		$mem->set("front_page", $images, time()+86400);
	}
}
else
{
	// This statement get the last 10 records from the database
	$images = retrieve_recent_uploads($db, 10);
}
```

Also pay attention to this code when doing image uploads. We deleted the cache after the user uploads an images. This way, when the next request comes in, we will fetch the latest records from the database, and put them into the cache again.

```php
	if ($enable_cache)
	{
		// Delete the cached record, the user will query the database to 
		// get an updated version
		$mem = open_memcache_connection($cache_server);
		$mem->delete("front_page");
	}
```