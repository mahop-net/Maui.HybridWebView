# Maui.HybridWebView
Maui sample project providing a WebView with JS -> C# and C# -> JS Interopt for building JS / HTML SPAs on all supported Maui platforms using C# as a "backend".

Small sample solution how you can call C# side from JS Code 
The other way around is supported natively by Maui WebView


Actually we use a little trick to do this:

- First we catch the Navigating event
- Than we get Data from the JS side
- Than we execute the CS Function


# C# Side

        public MainPage() {
            InitializeComponent();
            WebBrowser.Navigating += WebBrowserNavigating;
        }

        private void WebBrowserNavigating(object sender, WebNavigatingEventArgs e) {
            if (e.Url.Contains("/api/")) {
                var dataId = e.Url.Substring(e.Url.IndexOf("/api/") + 5);
                e.Cancel = true;
                Task.Run(() => {
                    Dispatcher.Dispatch(async () => {
                        var data = await WebBrowser.EvaluateJavaScriptAsync($"getData({dataId})");
                        var cdData = JsonSerializer.Deserialize<CsData>(data.Replace("\\", ""));

                        //Now Call the corresponding C# Funktion with data
                        // - this is only a simple example
                        // - you could also use a ICommand Interface, create an instance of a class implementing ICommand dynamically from csData.command and call it's execute Method, or ...
                        if (cdData != null) {
                            switch (cdData.command) {
                                case "Debug.WriteLine":
                                    Debug.WriteLine(cdData.data);

                                    //Just for the Show...
                                    await WebBrowser.EvaluateJavaScriptAsync($"log('{cdData.command} with dataId {cdData.dataId} was executed...')");
                                    break;
                            }
                        }
                    });
                });
            }
        }

# HTML

            <h1>Maui SPA</h1>
            <br />
            <p>
                <button onclick="callCs()">Call C#</button>
            </p>

# JS

       function callCs() {
            invokeCs("Debug.WriteLine", "Hello from Js!");
        }


        // Don't pollute the global namespace, but for the sake of simplicity for this example...
        let _data = [];
        let _dataId = 0;

        //Registers the data and calls the C# side
        function invokeCs(command, data) {
            //C# is calling getData async, so we have to provide a kind of data Store to be kind of "thread save"
            _dataId++;
            if (_dataId > 10000) {
                _dataId = 1;
            }
            const dataId = _dataId;

            const csData = new CsData(dataId, command, data);
            _data.push(csData);

            //Call the C# side - on the C# side the navigation is canceled but the command call will be executed after getting the data with dataId
            // You can also send smaller amounts of data directly here in the url, but there is a limit al little above 50K (URL should stay below 50K)
            window.location = "/api/" + dataId;
        }

        //Returns the Data for the command
        // - I did not test the limit here but 5 MB was no problem on Windows and Android
        function getData(dataId) {
            const data = _data.find(i => i.dataId === dataId);
            _data.splice(_data.indexOf(data), 1);
            return data;
        }

        function log(data) {
            console.log(data);
        }

        class CsData {
            constructor(dataId, command, data) {
                this.dataId = dataId;
                this.command = command;
                this.data = data;
            }
        }
