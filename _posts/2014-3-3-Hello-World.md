---
layout: post
title: NLog config in multiple environments deployment
---

Nlog doesn't support multiple environments config files. But we can reconfig it during run time based on the current asp.net core environment setting.
- Add settings to asp.net core environment specific congig files(appsettings.Production.json):
```
 "Logging": {
    "NLog": {
      "LogLevels": {
        "Default": "Error",
        "Sql": "Error"
      }
    }
  }
```
- Read settings and reconfig in Startup.cs
```
       public void UpdateLogLevel()
        {
            try
            {
                string logLevelSettingPrefix = KeySettingLogPrefix;
                LogManager.Configuration = LogManager.Configuration.Reload();
                var logLevels = _configuration.GetSection("Logging:NLog:LogLevels").GetChildren();
                foreach (var item in logLevels)
                {
                    if (!string.IsNullOrWhiteSpace(item.Value))
                        ChangeLogLevel(item.Key, item.Value);
                }
                var settings = Settings;
                var logLevelSettings = settings.AsQueryable().Where(x => x.Name.Trim().ToLower().StartsWith(KeySettingLogPrefix)).ToList();
                LogManager.ReconfigExistingLoggers();
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, string.Empty);
            }
        }
        private void ChangeLogLevel(string ruleName, string logLevel)
        {
            if (string.IsNullOrWhiteSpace(ruleName))
            {
                _logger.LogError("Rule name cannot be empty.");
                return;
            }
            if (string.IsNullOrWhiteSpace(logLevel))
            {
                _logger.LogError("Loglevel cannot be empty.");
                return;
            }
            NLog.LogLevel level = null;
            try
            {
                level = NLog.LogLevel.FromString(logLevel);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, $"Invalid Loglevel {logLevel}.");
                return;
            }
            var rules = LogManager.Configuration.LoggingRules;//.Where(x => x.NameMatches(loggerName));
            foreach (var rule in rules)
            {
                if (rule.RuleName?.ToLower() == ruleName.ToLower())
                {
                    rule.SetLoggingLevels(level, NLog.LogLevel.Fatal);
                    //// Iterate over all levels up to and including the target, (re)enabling them.
                    //for (int i = level.Ordinal; i <= 5; i++)
                    //{
                    //    rule.EnableLoggingForLevel(LogLevel.FromOrdinal(i));
                    //}
                }
            }
        }
        ...
        public void ConfigureServices(IServiceCollection services)
        {
          UpdateLogLevel();
        }
```
