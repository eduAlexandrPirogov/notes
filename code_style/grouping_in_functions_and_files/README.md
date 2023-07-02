# Группировка в функциях и файлах

Пример 1

В данном примере в блок помещено установка соответствия пользовательских типов данных к типам данным из библиотеки runtime. Выносить подобную установку не хотелось бы в отдельную функция, 
т.к. она нигде не будет использоваться помимо этого метода, а также не является достаточно автономной. Хоть пример и довольно простой, но он идеально подходит для демнострации данного метода.

```go
func (m *MemStats) Read() error {
	var runtimeMemStat runtime.MemStats
	runtime.ReadMemStats(&runtimeMemStat)

        // SEtting corresponding custom data types to runtime datatypes
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

==============================================================================================================================================================================================
==============================================================================================================================================================================================

Пример 2

В С++ подобный прием хорошо работает, поскольку часто приходится работать с библиотеками, где нужно устанавливать параметры через каждый вызов функции, как например libcurl:
```cpp

void RequestHandler::handleRequest(class GetCommand* command)
{
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

	// Additional request options 
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
    auto curl = curl_easy_init();
    std::string dropboxPath = dropboxPathPutCommand(command->dstPath());
    std::cout << command->srcPath() << '\n';
    if (curl) {
        
        //Security issue
        {
            headers = curl_slist_append(headers, authorizationCurlHeader().c_str());
            headers = curl_slist_append(headers, dropboxPath.c_str());
            headers = curl_slist_append(headers, "Content-Type: application/octet-stream");
        }
       
        // Additional request options 
        {
            curl_easy_setopt(curl, CURLOPT_URL, dropboxUrls->find(DROPBOX_API_URLS::UPLOAD_FILE));
            curl_easy_setopt(curl, CURLOPT_HTTPAUTH, (long)CURLAUTH_BEARER);
            curl_easy_setopt(curl, CURLOPT_CUSTOMREQUEST, "POST");
            curl_easy_setopt(curl, CURLOPT_TCP_KEEPALIVE, 1L);
            curl_easy_setopt(curl, CURLOPT_POST, 1L);
            curl_easy_setopt(curl, CURLOPT_HTTPHEADER, headers);
            curl_easy_setopt(curl, CURLOPT_READDATA, fp);
        }
        executeCurl(curl);
    }
};
```

Выносить в отдельные функции curl_slist_append и curl_easy_setopt нельзя, т.к. придется создавать метод с множеством параметров и функция будет не автономна. Подобный прием позволит 
"схлопнуть" установку параметров для curl'a, и упростить чтение кода.

==============================================================================================================================================================================================
==============================================================================================================================================================================================

Пример 3

Ещё одна область, где данный прием может использоваться -- создание desktop GUI, где зачастую для "отрисовки" объекта требуется установить различные параметры:

```cpp
GraphBuilderParamCtrlsPanel::GraphBuilderParamCtrlsPanel(wxWindow* param, std::vector<Json*>& query_result, GraphBuilderPanel* graphMainPanel)
	: wxPanel(param, wxID_ANY),graphMainPanel(graphMainPanel), query_result(query_result)
{
        // creating Button for creating graph
        {
	     create_graph = new wxButton(this, 30001, "Ïîêàçàòü", wxPoint(15, 15));
	     x_param = new wxComboBox(this, 30002, wxEmptyString, wxPoint(15, 50));
	     y_param = new wxComboBox(this, 30003, wxEmptyString, wxPoint(15, 100));
	}

        // creating RadioBox menu
        {
             wxArrayString choices;
	     choices.Add("Òî÷å÷íûé");
	     choices.Add("Ëèíèè");
	     choices.Add("Ñòîëáöû");
	     graph_type = new wxRadioBox(this, 30004, "Òèï ãðàôèêà", wxPoint(15, 150), wxDefaultSize, choices);
        } 
	
        // creating text areas
        {
	    equals_to = new wxTextCtrl(this, 30006, wxEmptyString, wxPoint(300, 15));*/
	    graph_title = new wxTextCtrl(this, 30006, "Ââåäèòå íàçâàíèå ãðàôèêà", wxPoint(175, 15));
        }
        
	// creating color picker
        {
            picker = new wxColourPickerCtrl(this, 30015, wxColour(150,150,150), wxPoint(100,15));
            sizer = new wxBoxSizer(wxVERTICAL);
	    sizer->Add(create_graph, 0, wxEXPAND | wxALL, 15);
	    sizer->Add(x_param, 0, wxEXPAND | wxALL, 15);
	    sizer->Add(y_param, 0, wxEXPAND | wxALL, 15);
	    sizer->Add(graph_type, 0, wxEXPAND | wxALL, 15);
	    sizer->Add(graph_title, 0, wxEXPAND | wxALL, 15);
	    sizer->Add(picker, 0, wxEXPAND | wxALL, 15);
	    SetSizerAndFit(sizer);
        }
}
```
Здесь мы помещаем в каждый блок отдельный объект для отрисовки. "схлопывание" позволит сократить количество кода для чтения, а обычно бОльшая часть кода для подобных приложений содержится в 
конструкторах, где происходит установки графических объектов, как например в данном случае на панели.

==============================================================================================================================================================================================
==============================================================================================================================================================================================

Итого -- просто хороший способ к структуризации кода. В ситуациях, где "кусок" не хотелось бы выносить в функции, этот прием заходит на ура :)
