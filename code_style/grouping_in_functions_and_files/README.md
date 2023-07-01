# Группировка в функциях и файлах

Пример 1

```go
func (m *MemStats) Read() error {
	var runtimeMemStat runtime.MemStats
	runtime.ReadMemStats(&runtimeMemStat)
	{
		m.Alloc = Alloc(runtimeMemStat.Alloc)
		m.BuckHashSys = BuckHashSys(runtimeMemStat.BuckHashSys)
		m.Frees = Frees(runtimeMemStat.Frees)
		m.GCCPUFraction = GCCPUFraction(runtimeMemStat.GCCPUFraction)
		m.GCSys = GCSys(runtimeMemStat.GCSys)
		m.HeapAlloc = HeapAlloc(runtimeMemStat.HeapAlloc)
		m.HeapIdle = HeapIdle(runtimeMemStat.HeapIdle)
		m.HeapInuse = HeapInuse(runtimeMemStat.HeapInuse)
		m.HeapObjects = HeapObjects(runtimeMemStat.HeapObjects)
		m.HeapReleased = HeapReleased(runtimeMemStat.HeapReleased)
		m.HeapSys = HeapSys(runtimeMemStat.HeapSys)
		m.LastGC = LastGC(runtimeMemStat.LastGC)
		m.Lookups = Lookups(runtimeMemStat.Lookups)
		m.MCacheInuse = MCacheInuse(runtimeMemStat.MCacheInuse)
		m.MCacheSys = MCacheSys(runtimeMemStat.MCacheSys)
		m.MSpanInuse = MSpanInuse(runtimeMemStat.MSpanInuse)
		m.MSpanSys = MSpanSys(runtimeMemStat.MSpanSys)
		m.Mallocs = Mallocs(runtimeMemStat.Mallocs)
		m.NextGC = NextGC(runtimeMemStat.NextGC)
		m.NumForcedGC = NumForcedGC(runtimeMemStat.NumForcedGC)
		m.NumGC = NumGC(runtimeMemStat.NumGC)
		m.OtherSys = OtherSys(runtimeMemStat.OtherSys)
		m.PauseTotalNs = PauseTotalNs(runtimeMemStat.PauseTotalNs)
		m.StackInuse = StackInuse(runtimeMemStat.StackInuse)
		m.StackSys = StackSys(runtimeMemStat.StackSys)
		m.Sys = Sys(runtimeMemStat.Sys)
		m.TotalAlloc = TotalAlloc(runtimeMemStat.TotalAlloc)
		m.RandomValue = RandomValue(rand.Float64())
	}
	return nil
}
```

Пример 2

```cpp

void RequestHandler::handleRequest(class GetCommand* command)
{
    if(!checkFileExistingOnDropbox(command->srcPath()))
    {
        std::string PathError = "\n\n\n+-------------ERROR!----------------+\n";
        PathError +=                  "|Given path not found on DropBox! :c|\n";
        PathError +=                  "+-----------------------------------+\n";
        throw std::runtime_error(PathError);
    }

    auto curl = curl_easy_init();
    string response_string;
    string header_string;
    std::string dropboxPath = dropboxPathGetCommand(command->srcPath());
    if (curl) {
        //file path
        fp = fopen(command->dstPath().c_str(), "wb");
        
 //Appending headers
	{
            headers = curl_slist_append(headers, authorizationCurlHeader().c_str());
            headers = curl_slist_append(headers, dropboxPath.c_str());
            headers = curl_slist_append(headers, "Content-Type: application/octet-stream");
        }

        {
            //setting curl options for correct request handle
            curl_easy_setopt(curl, CURLOPT_URL, dropboxUrls->find(DROPBOX_API_URLS::DOWNLOAD_FILE)); 
            //curl_easy_setopt(curl, CURLOPT_HTTPAUTH, (long)CURLAUTH_BEARER);
            curl_easy_setopt(curl, CURLOPT_CUSTOMREQUEST, "POST");
            curl_easy_setopt(curl, CURLOPT_FOLLOWLOCATION, 1);
            curl_easy_setopt(curl, CURLOPT_NOPROGRESS, 1L);
            curl_easy_setopt(curl, CURLOPT_MAXREDIRS, 50L);
            curl_easy_setopt(curl, CURLOPT_TCP_KEEPALIVE, 1L);
            curl_easy_setopt(curl, CURLOPT_HTTPHEADER, headers);

            curl_easy_setopt(curl, CURLOPT_WRITEDATA, fp);
        }
        executeCurl(curl);
        fclose(fp);
    }
};

void RequestHandler::handleRequest(class PutCommand* command)
{

    std::string givenPath = std::filesystem::path(command->dstPath()).remove_filename();

    if(!checkFileExistingOnDropbox(givenPath.substr(0, givenPath.size()-1)))
    {
        std::string PathError = "\n\n\n+-------------ERROR!----------------+\n";
        PathError +=                  "|Given path not found on DropBox! :c|\n";
        PathError +=                  "+-----------------------------------+\n";
        throw std::runtime_error(PathError);
    }

    auto curl = curl_easy_init();
    std::string dropboxPath = dropboxPathPutCommand(command->dstPath());
    std::cout << command->srcPath() << '\n';
    if (curl) {
        fp = fopen(command->srcPath().c_str(), "rb");

         if(fstat(fileno(fp), &file_info) != 0)
            throw std::runtime_error("Please, set your Access Code.\n You can get your code here: https://www.dropbox.com/oauth2/authorize?client_id=sv55ql8k090jinc&response_type=code\n");/* cannot continue */
       //Security issue
        headers = curl_slist_append(headers, authorizationCurlHeader().c_str());
        headers = curl_slist_append(headers, dropboxPath.c_str());
        headers = curl_slist_append(headers, "Content-Type: application/octet-stream");

        curl_easy_setopt(curl, CURLOPT_URL, dropboxUrls->find(DROPBOX_API_URLS::UPLOAD_FILE));
        curl_easy_setopt(curl, CURLOPT_HTTPAUTH, (long)CURLAUTH_BEARER);
        curl_easy_setopt(curl, CURLOPT_CUSTOMREQUEST, "POST");
        curl_easy_setopt(curl, CURLOPT_TCP_KEEPALIVE, 1L);
        curl_easy_setopt(curl, CURLOPT_POST, 1L);
        curl_easy_setopt(curl, CURLOPT_HTTPHEADER, headers);
        curl_easy_setopt(curl, CURLOPT_READDATA, fp);

        executeCurl(curl);
        fclose(fp);
    }
};
```
Пример 3
