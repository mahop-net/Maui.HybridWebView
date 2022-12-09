# Maui.HybridWebView
Maui sample project providing a WebView with JS -> C# and C# -> JS Interopt for building SPAs on all supported Maui platforms.

Small sample solution how you can call C# side from JS Code 
The other way around is supported natively by Maui WebView


Actually we use a little trick to do this:

- First we catch the Navigating event
- Than we get Data from the JS side
- Than we execute the CS Function

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
                        // - you could also get a kind of ICommand, create an instance of a class implementing ICommand dynamically from csData.command and call it's execute Method, or ...
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
