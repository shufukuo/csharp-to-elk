# Convert C# DataTable to Elastic bulk syntax
If you are new to C# or ELK, and you have to load data from C# to ELK, I hope this is helpful. Moreover, if C# NEST or Elasticsearch.Net is not available to you, here is an HTTP request example for your reference.
```csharp
using System;
using System.Linq;
using System.Text;
using System.Data;
using System.Net;
using System.IO;
using Newtonsoft.Json;


namespace csharp2elk
{
    class Program
    {
        static void Main(string[] args)
        {
            //
            // create example datatable
            //
            DataTable dt = new DataTable();
            dt.Columns.Add("C1", typeof(string));
            dt.Columns.Add("C2", typeof(int));
            dt.Columns.Add("C3", typeof(double));

            for (int i = 1; i <= 10; i++)
            {
                DataRow row = dt.NewRow();
                row["C1"] = "ID_" + i.ToString();
                row["C2"] = i;
                row["C3"] = (double)10 / i;
                dt.Rows.Add(row);
            }

            //
            // datatable2elk
            //  dt: datatable to input
            //  col_id: column that contains elk _id values
            //  rm_col_id: is col_id to be removed from dt
            //  elk_index: elk index name
            //  elk_type: elk type name
            //
            string bulk = datatable2elk(
                dt: dt,
                col_id: "C1",
                rm_col_id: false,
                elk_index: "myindex",
                elk_type: "mytype"
            );

            //
            // elk_summit example
            //
            string elk_return = elk_summit(
                elk_cmd: bulk,
                host: "http://127.0.0.1:9200",
                username: "username",
                password: "password",
                summit_method: "/_bulk?pretty"
            );

            return;
        }


        static string datatable2elk(DataTable dt, string col_id, bool rm_col_id, string elk_index, string elk_type)
        {
            //
            // initialize returned string
            //
            string res = "";

            //
            // declare elk bulk string
            // if _id not exists, create an _id and update doc
            // if _id exists, update doc
            //
            string str_bulk = @"
{ ""update"": { ""_index"": ""$_index$"", ""_type"": ""$_type$"", ""_id"": ""$_id$"" } }
{ ""script"": ""ctx.op='none'"", ""upsert"": { ""$col_upsert_name$"": null } }
{ ""update"": { ""_index"": ""$_index$"", ""_type"": ""$_type$"", ""_id"": ""$_id$"" } }
{ ""doc"": $doc$ }
"
                .Replace("$_index$", elk_index)
                .Replace("$_type$", elk_type)
                .Replace("$col_upsert_name$", dt.Columns[0].ColumnName)
            ;

            //
            // convert column names of dt to lowercase
            //
            foreach (DataColumn col in dt.Columns)
            {
                col.ColumnName = col.ColumnName.ToLower();
            }

            //
            // save elk _id values from dt, and delete column col_id if rm_col_id = true
            //
            col_id = col_id.ToLower();
            string[] values_id = dt.AsEnumerable().Select(x => x.Field<string>(col_id)).ToArray();

            if (rm_col_id)
            {
                dt.Columns.Remove(col_id);
            }

            //
            // convert each row to bulk syntax, and concatenate
            //
            StringBuilder sb_bulk = new StringBuilder("");

            StringBuilder sb_dt_json = new StringBuilder(
                JsonConvert.SerializeObject(dt).Replace("[", "").Replace("]", "")
            );

            for (var i = 0; i < dt.Rows.Count; i++)
            {
                var idx = 0;
                var tmp_ch = new char[3];

                while (true)
                {
                    if (idx <= sb_dt_json.Length - 3)
                    {
                        sb_dt_json.CopyTo(idx, tmp_ch, 0, 3);
                        if (new string(tmp_ch) == "},{")
                        {
                            break;
                        }
                    }
                    else
                    {
                        idx = -1;
                        break;
                    }
                    idx++;
                }

                if (idx > 0)
                {
                    tmp_ch = new char[idx + 1];
                    sb_dt_json.CopyTo(0, tmp_ch, 0, idx + 1);

                    sb_bulk.Append(new StringBuilder(str_bulk)
                        .Replace("$_id$", values_id[i])
                        .Replace("$doc$", new string(tmp_ch))
                    );

                    sb_dt_json.Remove(0, idx + 2);
                }
                else
                {
                    sb_bulk.Append(new StringBuilder(str_bulk)
                        .Replace("$_id$", values_id[i])
                        .Replace("$doc$", sb_dt_json.ToString())
                    );
                }
            }

            res = sb_bulk.ToString();

            return res;
        }


        static string elk_summit(string elk_cmd, string host, string username, string password, string summit_method, string requst_method = "POST")
        {
            string url = host + summit_method;

            byte[] contentbytes = Encoding.UTF8.GetBytes(elk_cmd);
            HttpWebRequest req = HttpWebRequest.Create(url) as HttpWebRequest;
            req.Method = requst_method;
            string auth = Convert.ToBase64String(ASCIIEncoding.ASCII.GetBytes(username + ":" + password));
            req.Headers.Add("Authorization", "Basic" + auth);
            req.ContentType = "application/x-ndjson";
            req.ContentLength = contentbytes.Length;

            try
            {
                using (Stream reqStream = req.GetRequestStream())
                {
                    reqStream.Write(contentbytes, 0, contentbytes.Length);
                }

                using (WebResponse res = req.GetResponse())
                {
                    using (StreamReader reader = new StreamReader(res.GetResponseStream()))
                    {
                        return reader.ReadToEnd();
                    }
                }
            }
            catch (Exception e)
            {
                return e.Message;
            }
        }
    }
}

```
