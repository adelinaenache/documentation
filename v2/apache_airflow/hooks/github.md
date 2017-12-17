---
title: Github Hook
sidebar: platform_sidebar
---

For a complete list of Airflow Hooks, Operators, and Utilities maintained by Astronomer, check out our [Airflow Plugins](https://github.com/airflow-plugins?utf8=%E2%9C%93&q=&type=&language=) organization on Github.

## Github

~~~ python
from airflow.hooks.http_hook import HttpHook


class GithubHook(HttpHook):

    def __init__(self, github_conn_id):
        self.github_token = None
        conn_id = self.get_connection(github_conn_id)
        if conn_id.extra_dejson.get('token'):
            self.github_token = conn_id.extra_dejson.get('token')
        super().__init__(method='GET', http_conn_id=github_conn_id)

    def get_conn(self, headers):
        """
        Accepts both Basic and Token Authentication.
        If a token exists in the "Extras" section
        with key "token", it is passed in the header.
        If not, the hook looks for a user name and password.
        In either case, it is important to ensure your privacy
        settings for your desired authentication method allow
        read access to user, org, and repo information.
        """
        if self.github_token:
            headers = {'Authorization': 'token {0}'.format(self.github_token)}
            session = super().get_conn(headers)
            session.auth = None
            return session
        return super().get_conn(headers)
~~~

## Google Analytics

~~~ python
from airflow.hooks.base_hook import BaseHook

from apiclient.discovery import build
from oauth2client.service_account import ServiceAccountCredentials

import time

class GoogleAnalyticsHook(BaseHook):
    def __init__(self, google_analytics_conn_id='google_analytics_default'):
        self.google_analytics_conn_id = google_analytics_conn_id
        self.connection = self.get_connection(google_analytics_conn_id)

        self.client_secrets = self.connection.extra_dejson['client_secrets']

    def get_service_object(self, api_name, api_version, scopes):
        credentials = ServiceAccountCredentials.from_json_keyfile_dict(self.client_secrets, scopes)
        return build(api_name, api_version, credentials=credentials)

    def get_analytics_report(self, view_id, since, until, sampling_level, dimensions, metrics, page_size, include_empty_rows):
        analytics = self.get_service_object('analyticsreporting', 'v4', ['https://www.googleapis.com/auth/analytics.readonly'])

        reportRequest = {
            'viewId': view_id,
            'dateRanges': [{ 'startDate': since, 'endDate': until }],
            'samplingLevel': sampling_level or 'LARGE',
            'dimensions': dimensions,
            'metrics': metrics,
            'pageSize': page_size or 100,
            'includeEmptyRows': include_empty_rows or False
        }

        response = analytics.reports().batchGet(body={ 'reportRequests': [reportRequest] }).execute()

        if response.get('reports'):
            report = response['reports'][0]
            rows = report.get('data', {}).get('rows', [])

            while report.get('nextPageToken'):
                time.sleep(1)
                reportRequest.update({ 'pageToken': report['nextPageToken'] })
                response = analytics.reports().batchGet(body={ 'reportRequests': [reportRequest] }).execute()
                report = response['reports'][0]
                rows.extend(report.get('data', {}).get('rows', []))

            if report['data']:
                report['data']['rows'] = rows

            return report
        else:
            return {}
~~~

[Source](https://github.com/airflow-plugins/google_analytics_plugin/blob/master/hooks/google_analytics_hook.py)