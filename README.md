# JsonConverter
Custom Json Converter using JsonSerializer (System.Text.Json)

I faced a scenario  where My Controller Action method was unable to parse the JSON received in Request body and map it to the Model.
The Action method looked like below
# Action Method
        public JsonResult SaveDigitalPromoPlannerOLD([FromBody] DigitalPromoPlannerViewModel digPlannerViewModel)
        {
            try
            {        

                DigitalPlannerBO bo = new DigitalPlannerBO(this.CurrentUser);
                bo.Load(digPlannerViewModel);
                bo.SaveDigitalPromoPlanner();
                return Json(bo.GetDigitalPlannerViewModel());//, JsonRequestBehavior.AllowGet);               

            }
            catch (Exception ex)
            {
                this.LogSkinnyFBError("SaveDigitalPromoPlanner", ex);
                return Json(new { status = "Fail", message = "Digital Promotion Planner Save Failed" });
            }
        }

  DigitalPromoPlannerViewModel has some Date and Decimal attributes but the values that we received in JSON were in string format.
  Date was received as "mm/dd/yyyy" and Decimal was received as "3.50". With default options of JsonSerializer, input DateTime text representations must conform to the extended ISO 8601-1:2019   
  profile so the model binding failed.

  How I overcame this problem is writing custom Json Converters and additing them as the options. I created once custom converter for DateTime annd Another for Decimal
# DateTime Converter
    public class DateTimeConverterUsingDateTimeParse : JsonConverter<DateTime>
    {
      public override DateTime Read(ref Utf8JsonReader reader, Type typeToConvert, JsonSerializerOptions options)
      {
          Debug.Assert(typeToConvert == typeof(DateTime));
          return DateTime.Parse(reader.GetString() ?? string.Empty);
      }
  
      public override void Write(Utf8JsonWriter writer, DateTime value, JsonSerializerOptions options)
      {
          writer.WriteStringValue(value.ToString());
      }
    }

# Decimal Converter
    public class DecimalConverterUsingDecimalParse: JsonConverter<Decimal>
    {
        public override Decimal Read(ref Utf8JsonReader reader, Type typeToConvert, JsonSerializerOptions options)
        {
            
            if (reader.TokenType == JsonTokenType.String)
            {
                return Decimal.Parse(reader.GetString() ?? string.Empty);
                //return reader.GetString();
            }
            else
            {
                return reader.GetDecimal();
            }     
    
        }
    
  
        public override void Write(Utf8JsonWriter writer, decimal value, JsonSerializerOptions options)
        {
            //throw new NotImplementedException();
            writer.WriteStringValue(value.ToString());
        }
    }


Then modified the controller action method like this to work

# Modified Action Method

    public async Task<JsonResult> SaveDigitalPromoPlanner()
            {
                try
                {
                   
                    var bodyStr = "";
                    var req = this.HttpContext.Request;                    
                    using (StreamReader reader  = new StreamReader(req.Body, Encoding.UTF8))
                    {
                        bodyStr = await reader.ReadToEndAsync();;
                    }                    
                    JsonSerializerOptions options = new JsonSerializerOptions();
                    options.Converters.Add(new DateTimeConverterUsingDateTimeParse());
                    options.Converters.Add(new DecimalConverterUsingDecimalParse());
                    DigitalPromoPlannerViewModel? digPlannerViewModel = 
                    JsonSerializer.Deserialize<DigitalPromoPlannerViewModel>(bodyStr,options);                
    
                    DigitalPlannerBO bo = new DigitalPlannerBO(this.CurrentUser);
                    bo.Load(digPlannerViewModel);
                    bo.SaveDigitalPromoPlanner();
                    return Json(bo.GetDigitalPlannerViewModel());//, JsonRequestBehavior.AllowGet);               
    
                }
                catch (Exception ex)
                {
                    this.LogSkinnyFBError("SaveDigitalPromoPlanner", ex);
                    return Json(new { status = "Fail", message = "Digital Promotion Planner Save Failed" });
                }
            }


We can add these Options to the JsonSelializer during dependency injection @Program.cs. That way we can modify the default Json Serlializer behavior and we dont have do serialization in every controler level.

        builder.Services.AddControllersWithViews()
                    .AddJsonOptions(options => 
                       options.JsonSerializerOptions.PropertyNamingPolicy = null)
                    .AddJsonOptions(options => 
                        options.JsonSerializerOptions.Converters.Add(new DateTimeConverterUsingDateTimeParse()))
                    .AddJsonOptions(options => 
                        options.JsonSerializerOptions.Converters.Add(new DecimalConverterUsingDecimalParse()));
