# Code Changes Summary

## File 1: AvailabilityRuleHelper.cs

### Added new BatchEpisodeCheck method
```csharp
/// <summary>
/// Optimized batch check for all episodes of a TV show.
/// Fetches all episodes for the series once, then matches them in-memory instead of per-episode queries.
/// </summary>
public static async Task BatchEpisodeCheck(bool useImdb, IQueryable<IMediaServerEpisode> allEpisodes, 
    List<SeasonRequests> seasonRequests, IMediaServerContent item, bool useTheMovieDb, bool useTvDb, ILogger log)
{
    try
    {
        // Fetch all episodes for this series at once
        IQueryable<IMediaServerEpisode> seriesEpisodes = null;

        if (useImdb && item.ImdbId.HasValue())
        {
            seriesEpisodes = allEpisodes.Where(x => x.Series.ImdbId == item.ImdbId);
        }
        else if (useTheMovieDb && item.TheMovieDbId.HasValue())
        {
            seriesEpisodes = allEpisodes.Where(x => x.Series.TheMovieDbId == item.TheMovieDbId);
        }
        else if (useTvDb && item.TvDbId.HasValue())
        {
            seriesEpisodes = allEpisodes.Where(x => x.Series.TvDbId == item.TvDbId);
        }

        if (seriesEpisodes != null)
        {
            // Convert to list once to avoid multiple database calls
            var episodeList = await seriesEpisodes.ToListAsync();

            // Now check each episode against the in-memory list
            foreach (var season in seasonRequests)
            {
                foreach (var episode in season.Episodes)
                {
                    var epExists = episodeList.FirstOrDefault(x =>
                        x.EpisodeNumber == episode.EpisodeNumber && x.SeasonNumber == season.SeasonNumber);

                    if (epExists != null)
                    {
                        episode.Available = true;
                    }
                }
            }
        }
    }
    catch (Exception e)
    {
        log.LogError(e, "Exception thrown when attempting to check if episodes are available");
    }
}
```

## File 2: JellyfinAvailabilityRule.cs

### Before
```csharp
if (obj.Type == RequestType.TvShow)
{
    var search = (SearchTvShowViewModel)obj;
    if (search.SeasonRequests.Any())
    {
        var allEpisodes = JellyfinContentRepository.GetAllEpisodes().Include(x => x.Series);
        foreach (var season in search.SeasonRequests)
        {
            foreach (var episode in season.Episodes)
            {
                // Calls database for EACH episode
                await AvailabilityRuleHelper.SingleEpisodeCheck(useImdb, allEpisodes, episode, season, item, useTheMovieDb, useTvDb, Log);
            }
        }
    }

    AvailabilityRuleHelper.CheckForUnairedEpisodes(search);
}
```

### After
```csharp
if (obj.Type == RequestType.TvShow)
{
    var search = (SearchTvShowViewModel)obj;
    if (search.SeasonRequests.Any())
    {
        var allEpisodes = JellyfinContentRepository.GetAllEpisodes().Include(x => x.Series);
        // Use batch check instead of per-episode queries
        await AvailabilityRuleHelper.BatchEpisodeCheck(useImdb, allEpisodes, search.SeasonRequests, item, useTheMovieDb, useTvDb, Log);
    }

    AvailabilityRuleHelper.CheckForUnairedEpisodes(search);
}
```

## File 3: EmbyAvailabilityRule.cs

### Before
```csharp
if (obj.Type == RequestType.TvShow)
{
    var search = (SearchTvShowViewModel)obj;
    if (search.SeasonRequests.Any())
    {
        var allEpisodes = EmbyContentRepository.GetAllEpisodes().Include(x => x.Series);
        foreach (var season in search.SeasonRequests)
        {
            foreach (var episode in season.Episodes)
            {
                // Calls database for EACH episode
                await AvailabilityRuleHelper.SingleEpisodeCheck(useImdb, allEpisodes, episode, season, item, useTheMovieDb, useTvDb, Log);
            }
        }
    }

    AvailabilityRuleHelper.CheckForUnairedEpisodes(search);
}
```

### After
```csharp
if (obj.Type == RequestType.TvShow)
{
    var search = (SearchTvShowViewModel)obj;
    if (search.SeasonRequests.Any())
    {
        var allEpisodes = EmbyContentRepository.GetAllEpisodes().Include(x => x.Series);
        // Use batch check instead of per-episode queries
        await AvailabilityRuleHelper.BatchEpisodeCheck(useImdb, allEpisodes, search.SeasonRequests, item, useTheMovieDb, useTvDb, Log);
    }

    AvailabilityRuleHelper.CheckForUnairedEpisodes(search);
}
```

## File 4: PlexAvailabilityRule.cs

### Before
```csharp
if (obj is SearchTvShowViewModel search)
{
    if (search.SeasonRequests.Any())
    {
        var allEpisodes = PlexContentRepository.GetAllEpisodes();
        foreach (var season in search.SeasonRequests.ToList())
        {
            foreach (var episode in season.Episodes.ToList())
            {
                // Calls database for EACH episode
                await AvailabilityRuleHelper.SingleEpisodeCheck(useImdb, allEpisodes, episode, season, item, useTheMovieDb, useTvDb, Log);
            }
        }

        AvailabilityRuleHelper.CheckForUnairedEpisodes(search);
    }
}
```

### After
```csharp
if (obj is SearchTvShowViewModel search)
{
    if (search.SeasonRequests.Any())
    {
        var allEpisodes = PlexContentRepository.GetAllEpisodes();
        // Use batch check instead of per-episode queries
        await AvailabilityRuleHelper.BatchEpisodeCheck(useImdb, allEpisodes, search.SeasonRequests, item, useTheMovieDb, useTvDb, Log);

        AvailabilityRuleHelper.CheckForUnairedEpisodes(search);
    }
}
```

## Key Improvements

1. **Eliminated N+1 Query Pattern**: Instead of 1 query per episode, now only 1 query per series
2. **In-Memory Matching**: After fetching episodes once, all matching is done in-memory (fast)
3. **No Breaking Changes**: Original `SingleEpisodeCheck()` method retained for backward compatibility
4. **Consistent Implementation**: Same optimization applied to all three media server types (Jellyfin, Emby, Plex)
