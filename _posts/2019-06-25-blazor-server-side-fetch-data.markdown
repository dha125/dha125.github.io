---
layout: post
title:  "Blazor Server-Side Fetch Data"
date:   2019-06-25 11:21:01 -0600
categories: .Net Core Blazor
---
I started by creating a new project by choosing **New Project** > **ASP.NET Core Web Application** > 
**Blazor (Server-side in ASP.NET Core)**. I wanted to see how robust this out of the box template code was. 
I added a `Thread.Sleep(1000);` in the WeatherForecastService below (Listing 1) to simulate a realistic 
network service. The code out of the template placed the call to WeatherForecastService in the 
`protected override async Task OnInitAsync()` method. The delay caused the page to be 
unresponsive and the message 'Loading...' never appeared for me. Additionally, I added a random exception 
to simulate a network disconnection or other serious event.

Comming from a Client/Server background I was looking for a better BackgroundWork pattern. In the meantime, I've 
moved the service call to `OnAfterRenderAsync` and added some exception handling and reload functionality (Listing 2).
Overall, I'm happy with the results now, but this still lacks a good BackgroundWork pattern and the elegancy of less 
code behind that we'd expect from in WPF or UWP and the MVVM pattern. 

Listing 1: WeatherForecastService
```c#
using System.Linq;
using System.Threading;
using System.Threading.Tasks;

namespace WebApplication1.Data
{
  public class WeatherForecastService
  {
    private static string[] Summaries = new[]
    {
            "Freezing", "Bracing", "Chilly", "Cool", "Mild", "Warm", "Balmy", 
			"Hot", "Sweltering", "Scorching"
        };

    public Task<WeatherForecast[]> GetForecastAsync(DateTime startDate)
    {
      var rng = new Random();

      Thread.Sleep(1000);

      if( rng.Next(10) > 5)
      {
        throw new Exception("Snap..."); 
      }

      return Task.FromResult(Enumerable.Range(1, 5).Select(index => 
	  new WeatherForecast
      {
        Date = startDate.AddDays(index),
        TemperatureC = rng.Next(-20, 55),
        Summary = Summaries[rng.Next(Summaries.Length)]
      }).ToArray());
    }

  }
}
```


Listing 2: Pages/FetchData.razor
```c#
@page "/fetchdata"
@using WebApplication1.Data
@inject WeatherForecastService ForecastService

<h1>Weather forecast</h1>

<p>This component demonstrates fetching data from a service.</p>

@if (forecasts == null)
{
    if (errorMessage == null)
    {
        <p><em>Loading...</em></p>
    }
    else
    {
        <button class="btn btn-primary" onclick="@Reload">Reload</button>
        <p>@errorMessage</p>
    }
}
else
{
    <button class="btn btn-primary" onclick="@Reload">Reload</button>
    <div></div>
    <table class="table">
        <thead>
            <tr>
                <th>Date</th>
                <th>Temp. (C)</th>
                <th>Temp. (F)</th>
                <th>Summary</th>
            </tr>
        </thead>
        <tbody>
            @foreach (var forecast in forecasts)
            {
                <tr>
                    <td>@forecast.Date.ToShortDateString()</td>
                    <td>@forecast.TemperatureC</td>
                    <td>@forecast.TemperatureF</td>
                    <td>@forecast.Summary</td>
                </tr>
            }
        </tbody>
    </table>
}

@code {
    WeatherForecast[] forecasts;

    bool once;

    string errorMessage = null;

    void Reload()
    {
        once = false;
        forecasts = null;
    }

    protected override void OnInit()
    {
        once = false;
        errorMessage = null;
        forecasts = null;
    }

    protected override async Task OnAfterRenderAsync()
    {
        if (!once)
        {
            try
            {
                forecasts = await ForecastService.GetForecastAsync(DateTime.Now);
            }
            catch
            {
                errorMessage = "There was a problem connecting with the forecast " +
                    "service. Please reload to try again.";
            }
            finally
            {
                once = true;
            }
            await Invoke(() =>
            {
                StateHasChanged();
            });
        }
    }

}
```
