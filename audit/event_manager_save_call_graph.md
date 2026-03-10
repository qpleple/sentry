# Complete Call Graph: `EventManager.save()`

**Source file:** `src/sentry/event_manager.py`
**Entry point:** `src/sentry/event_manager.py:EventManager.save` (line 439)

## Format

```
caller -> callee [condition]
```

- `[condition]` describes when the edge is taken
- `(DYNAMIC)` marks calls resolved at runtime via registries, settings, or signals
- `(SIGNAL)` marks Django signal dispatch with multiple receivers
- `(CELERY)` marks async Celery task dispatch
- `(KAFKA)` marks Kafka message production
- `(BACKEND)` marks calls to configurable backends resolved via `LazyServiceWrapper` / `import_string`

Function IDs use the format: `relative/path.py:ClassName.method` or `relative/path.py:function_name`

---

## 1. Root: `EventManager.save()`

```
src/sentry/event_manager.py:EventManager.save
  -> src/sentry/event_manager.py:resolve_project [project is None]
       -> src/sentry/models/project.py:Project.objects.get_from_cache
       -> src/sentry/models/organization.py:Organization.objects.get_from_cache
       -> src/sentry/models/project.py:Project.set_cached_field_value
  -> src/sentry/event_manager.py:EventManager.normalize [not self._normalized and not assume_normalized]
       -> src/sentry/event_manager.py:EventManager._normalize_impl
  -> src/sentry/event_manager.py:_pull_out_data
       -> src/sentry/event_manager.py:_get_event_instance
            -> src/sentry/eventstore/__init__.py:backend.create_event (BACKEND)
                 -> src/sentry/eventstore/snuba/backend.py:SnubaEventStorage.create_event
                      -> src/sentry/eventstore/models.py:Event.__init__
       -> src/sentry/quotas/base.py:index_data_category
       -> src/sentry/event_manager.py:set_tag [multiple calls]
       -> src/sentry/utils/safe.py:setdefault_path
       -> django.utils.encoding:force_str
  -> src/sentry/event_manager.py:_set_project_platform_if_needed
       -> src/sentry/eventstore/models.py:Event.get_tag ["sample_event" check]
       -> src/sentry/models/project.py:Project.objects.filter().update() [platform not set]
       -> src/sentry/utils/audit.py:create_system_audit_entry [platform updated]
            -> src/sentry/audit_log/__init__.py:audit_log.get_event_id
            -> src/sentry/models/auditlogentry.py:AuditLogEntry.__init__
            -> src/sentry/audit_log/services/log.py:log_service.record_audit_log
  -> src/sentry/event_manager.py:save_transaction_events [event_type == "transaction"]
  -> src/sentry/event_manager.py:save_generic_events [event_type == "generic"]
  -> src/sentry/grouping/ingest/config.py:is_in_transition [populates metric_tags]
  -> src/sentry/grouping/enhancer/__init__.py:get_enhancements_version [populates metric_tags]
  -> src/sentry/utils/tag_normalization.py:normalized_sdk_tag_from_event [populates metric_tags]
  -> src/sentry/event_manager.py:EventManager.save_error_events [else (error/default/csp/etc.)]
```

---

## 2. Error Path: `EventManager.save_error_events()`

```
src/sentry/event_manager.py:EventManager.save_error_events
  -> src/sentry/reprocessing2/__init__.py:is_reprocessed_event
  -> src/sentry/event_manager.py:_get_or_create_release_many
       [returns early if no release in data]
       -> src/sentry/models/release.py:Release.get_or_create
            -> src/sentry/models/release.py:Release.objects.create [try]
            -> src/sentry/models/release.py:Release.objects.get [on IntegrityError]
            -> src/sentry/models/release.py:Release.add_project
                 -> src/sentry/models/releases/release_project.py:ReleaseProject.objects.get_or_create
       [catches ValidationError -> sets release=None]
       -> src/sentry/event_manager.py:pop_tag [remove conflicting "release" tag]
       -> src/sentry/event_manager.py:set_tag ["sentry:release"]
       -> src/sentry/models/release.py:Release.add_dist [if dist in data]
       -> src/sentry/event_manager.py:pop_tag [remove conflicting "dist" tag]
       -> src/sentry/event_manager.py:set_tag ["sentry:dist"]
  -> src/sentry/event_manager.py:_get_event_user_many
       -> src/sentry/event_manager.py:_get_event_user
            -> src/sentry/event_manager.py:_get_event_user_impl
                 -> src/sentry/utils/eventuser.py:EventUser.__init__
       -> src/sentry/event_manager.py:pop_tag [remove "user" tag]
       -> src/sentry/event_manager.py:set_tag ["sentry:user"]
  -> src/sentry/models/projectkey.py:ProjectKey.objects.get_from_cache [key_id is not None]
  -> src/sentry/event_manager.py:_derive_plugin_tags_many
       -> src/sentry/plugins/base/manager.py:PluginManager.for_project (DYNAMIC)
            -> src/sentry/plugins/base/v1.py:Plugin.is_enabled [for each registered plugin]
       -> src/sentry/plugins/base/v1.py:Plugin.get_tags (DYNAMIC) [for each enabled plugin]
       -> src/sentry/event_manager.py:get_tag [check if plugin tag already exists]
       -> src/sentry/event_manager.py:set_tag [add plugin tags]
            -> src/sentry/event_manager.py:pop_tag [remove existing tag before re-setting]
  -> src/sentry/event_manager.py:_derive_interface_tags_many
       -> src/sentry/eventstore/models.py:Event.interfaces [property]
       -> src/sentry/interfaces/base.py:Interface.iter_tags (DYNAMIC) [for each interface]
  -> src/sentry/event_manager.py:_derive_client_error_sampling_rate
  -> src/sentry/event_manager.py:assign_event_to_group
       [SEE SECTION 5 - GROUPING SUBTREE]
  -> src/sentry/event_manager.py:increment_group_tombstone_hit_counter [on HashDiscarded exception]
       [returns early if tombstone_id is None]
       -> src/sentry/models/grouptombstone.py:GroupTombstone.objects.get
       -> src/sentry/tasks/process_buffer.py:buffer_incr [GroupTombstone] (BACKEND)
       [catches GroupTombstone.DoesNotExist (race with deletion)]
  -> src/sentry/event_manager.py:discard_event [on HashDiscarded exception]
       [SEE SECTION 9 - DISCARD/ATTACHMENTS]
  [returns early if group_info is None - event not assigned to any group]
  -> src/sentry/eventstore/models.py:EventDict.bind_ref [store group_id reference]
  -> src/sentry/event_manager.py:_get_or_create_environment_many
       -> src/sentry/models/environment.py:Environment.get_or_create
  -> src/sentry/event_manager.py:_get_or_create_group_environment_many
       -> src/sentry/event_manager.py:_get_or_create_group_environment
            -> src/sentry/models/groupenvironment.py:GroupEnvironment.get_or_create
  -> src/sentry/event_manager.py:_get_or_create_release_associated_models
       -> src/sentry/models/releaseenvironment.py:ReleaseEnvironment.get_or_create
       -> src/sentry/models/releaseprojectenvironment.py:ReleaseProjectEnvironment.get_or_create
  -> src/sentry/event_manager.py:_increment_release_associated_counts_many
       -> src/sentry/event_manager.py:_increment_release_associated_counts
            -> src/sentry/tasks/process_buffer.py:buffer_incr [ReleaseProject] (BACKEND)
                 -> src/sentry/buffer/redis.py:RedisBuffer.incr [default backend]
                      -> Redis HINCRBY / pipeline operations
            -> src/sentry/tasks/process_buffer.py:buffer_incr [ReleaseProjectEnvironment] (BACKEND)
  -> src/sentry/event_manager.py:_get_or_create_group_release_many
       -> src/sentry/event_manager.py:_get_or_create_group_release
            -> src/sentry/models/grouprelease.py:GroupRelease.get_or_create
  -> src/sentry/event_manager.py:_tsdb_record_all_metrics
       [SEE SECTION 10 - TSDB]
  -> src/sentry/event_manager.py:filter_attachments_for_group [if attachments]
       [SEE SECTION 9 - DISCARD/ATTACHMENTS]
  -> src/sentry/event_manager.py:_materialize_event_metrics
  -> src/sentry/event_manager.py:_nodestore_save_many
       [SEE SECTION 11 - NODESTORE]
  -> src/sentry/models/project.py:Project.update [not raw and not project.first_event]
  -> src/sentry/signals:first_event_received.send_robust [not raw and not project.first_event] (SIGNAL)
       -> src/sentry/receivers/features.py:record_first_event
            -> src/sentry/models/featureadoption.py:FeatureAdoption.objects.record()
       -> src/sentry/receivers/onboarding.py:record_first_event
            -> src/sentry/models/organizationonboardingtask.py:OrganizationOnboardingTask update
            -> src/sentry/analytics/__init__.py:analytics.record [FirstEventSentEvent]
  -> src/sentry/utils/event.py:has_event_minified_stack_trace [not raw]
  -> src/sentry/utils/projectflags.py:set_project_flag_and_signal [has minified stack trace]
       -> src/sentry/models/project.py:Project.update [flags bitor]
       -> src/sentry/signals:first_event_with_minified_stack_trace_received.send_robust (SIGNAL)
            -> src/sentry/receivers/onboarding.py:record_event_with_first_minified_stack_trace_for_project
  -> src/sentry/reprocessing2/__init__.py:buffered_delete_old_primary_hash [is_reprocessed] (via safe_execute)
  -> src/sentry/event_manager.py:_eventstream_insert_many
       [SEE SECTION 12 - EVENTSTREAM]
  -> src/sentry/event_manager.py:save_attachments [not is_reprocessed and attachments]
       [SEE SECTION 9 - DISCARD/ATTACHMENTS]
  -> src/sentry/event_manager.py:_track_outcome_accepted_many
       -> src/sentry/utils/outcomes.py:OutcomeAggregator.track_outcome_aggregated
            -> Buffers outcome, periodically flushes to track_outcome
                 -> src/sentry/utils/outcomes.py:track_outcome
                      -> Kafka produce to outcomes topic
```

---

## 3. Transaction Path: `save_transaction_events()`

```
src/sentry/event_manager.py:save_transaction_events
  -> src/sentry/models/organization.py:Organization.objects.get_many_from_cache
  -> src/sentry/models/project.py:Project.set_cached_field_value
  -> src/sentry/event_manager.py:_get_or_create_release_many [same as error path]
  -> src/sentry/event_manager.py:_get_event_user_many [same as error path]
  -> src/sentry/event_manager.py:_derive_plugin_tags_many [same as error path]
  -> src/sentry/event_manager.py:_derive_interface_tags_many [same as error path]
  -> src/sentry/event_manager.py:_calculate_span_grouping
       -> src/sentry/eventstore/models.py:Event.get_span_groupings
       -> SpanGroupingResults.write_to_event
  -> src/sentry/event_manager.py:_materialize_metadata_many
       -> src/sentry/event_manager.py:get_event_type
            -> src/sentry/eventtypes/__init__.py:eventtypes.get (DYNAMIC registry)
                 -> Returns one of: DefaultEvent, ErrorEvent, TransactionEvent, CspEvent, etc.
       -> EventType.get_metadata [polymorphic on event type]
       -> src/sentry/event_manager.py:materialize_metadata
            -> src/sentry/event_manager.py:get_culprit
                 -> src/sentry/culprit.py:generate_culprit
            -> EventType.get_title [polymorphic]
            -> EventType.get_location [polymorphic]
  -> src/sentry/event_manager.py:_get_or_create_environment_many [same as error path]
  -> src/sentry/event_manager.py:_get_or_create_release_associated_models [same as error path]
  -> src/sentry/event_manager.py:_tsdb_record_all_metrics [SEE SECTION 10]
  -> src/sentry/event_manager.py:_materialize_event_metrics
  -> src/sentry/event_manager.py:_nodestore_save_many [SEE SECTION 11]
  -> src/sentry/event_manager.py:_eventstream_insert_many [SEE SECTION 12]
  -> src/sentry/utils/event_tracker.py:track_sampled_event
  -> src/sentry/event_manager.py:_track_outcome_accepted_many [same as error path]
  -> src/sentry/event_manager.py:_detect_performance_problems
       -> src/sentry/issue_detection/performance_detection.py:detect_performance_problems
            -> src/sentry/issue_detection/performance_detection.py:_detect_performance_problems
                 -> Runs multiple performance detectors (N+1 queries, slow DB, etc.)
  -> src/sentry/event_manager.py:_send_occurrence_to_platform
       -> src/sentry/issues/issue_occurrence.py:IssueOccurrence.__init__ [for each perf problem]
       -> src/sentry/issues/producer.py:produce_occurrence_to_kafka (KAFKA)
            -> Kafka produce to Topic.INGEST_OCCURRENCES
  -> src/sentry/event_manager.py:_record_transaction_info
       -> src/sentry/ingest/transaction_clusterer/datasource/redis.py:record_transaction_name
            -> Redis SADD (capped set via Lua script)
       -> src/sentry/receivers/features.py:record_event_processed
            -> src/sentry/models/featureadoption.py:FeatureAdoption.objects.bulk_record
       -> src/sentry/utils/projectflags.py:set_project_flag_and_signal ["has_transactions"] [not skip_send_first_transaction]
            -> src/sentry/signals:first_transaction_received.send_robust (SIGNAL)
                 -> src/sentry/receivers/onboarding.py:record_first_transaction
       -> src/sentry/insights/__init__.py:FilterSpan.from_span_v1 [per span]
       -> src/sentry/insights/__init__.py:modules [filter spans by module]
       -> src/sentry/utils/projectflags.py:set_project_flag_and_signal [per insight module] (SIGNAL)
            -> src/sentry/signals:first_insight_span_received.send_robust
                 -> src/sentry/receivers/onboarding.py:record_first_insight_span
       -> src/sentry/dynamic_sampling/rules/helpers/latest_releases.py:record_latest_release [if release]
            -> Redis operations (boosted releases tracking)
            -> src/sentry/tasks/relay.py:schedule_invalidate_project_config (CELERY) [on new boost]
```

---

## 4. Generic Path: `save_generic_events()`

```
src/sentry/event_manager.py:save_generic_events
  -> src/sentry/models/organization.py:Organization.objects.get_many_from_cache
  -> src/sentry/models/project.py:Project.set_cached_field_value
  -> src/sentry/event_manager.py:_get_or_create_release_many [same as error path]
  -> src/sentry/event_manager.py:_get_event_user_many [same as error path]
  -> src/sentry/event_manager.py:_derive_plugin_tags_many [same as error path]
  -> src/sentry/event_manager.py:_derive_interface_tags_many [same as error path]
  -> src/sentry/event_manager.py:_materialize_metadata_many [same as transaction path]
  -> src/sentry/event_manager.py:_get_or_create_environment_many [same as error path]
  -> src/sentry/event_manager.py:_materialize_event_metrics
  -> src/sentry/event_manager.py:_nodestore_save_many [SEE SECTION 11]
```

---

## 5. Grouping Subtree: `assign_event_to_group()`

```
src/sentry/event_manager.py:assign_event_to_group
  -> src/sentry/event_manager.py:get_hashes_and_grouphashes [primary]
       -> src/sentry/grouping/ingest/hashing.py:run_primary_grouping (callback)
            -> src/sentry/grouping/api.py:get_grouping_config_dict_for_project
            -> src/sentry/grouping/ingest/hashing.py:_calculate_primary_hashes_and_variants
                 -> src/sentry/grouping/api.py:get_grouping_variants_for_event
       -> src/sentry/grouping/ingest/hashing.py:get_or_create_grouphashes
            -> src/sentry/grouping/ingest/hashing.py:_get_or_create_single_grouphash
                 -> src/sentry/models/grouphash.py:GroupHash.objects.get_or_create
            -> src/sentry/grouping/ingest/grouphash_metadata.py:create_or_update_grouphash_metadata_if_needed
            -> src/sentry/grouping/ingest/grouphash_metadata.py:record_grouphash_metadata_metrics
       -> src/sentry/grouping/ingest/hashing.py:find_grouphash_with_group
            -> src/sentry/models/grouphash.py:GroupHash [filters for group_id not null]
            -> raises HashDiscarded [if tombstone found]
  -> src/sentry/event_manager.py:handle_existing_grouphash [primary.existing_grouphash found]
       [SEE SECTION 6]
  -> src/sentry/grouping/ingest/seer.py:maybe_send_seer_for_new_model_training [primary existing grouphash found]
       -> src/sentry/seer/similarity/config.py:should_send_to_seer_for_training
       -> src/sentry/grouping/ingest/seer.py:should_call_seer_for_grouping
       -> src/sentry/grouping/ingest/seer.py:_build_seer_request
       -> src/sentry/seer/signed_seer_api.py:SeerViewerContext.__init__
       -> src/sentry/seer/similarity/similar_issues.py:get_similarity_data_from_seer
            -> HTTP POST to Seer service /v0/issues/similar-issues
       -> src/sentry/seer/similarity/config.py:get_new_model_version
       -> GroupHashMetadata.update [seer_latest_training_model]
  -> src/sentry/event_manager.py:get_hashes_and_grouphashes [secondary, if no primary match]
       -> src/sentry/grouping/ingest/hashing.py:maybe_run_secondary_grouping (callback)
            -> src/sentry/grouping/ingest/config.py:is_in_transition
            -> src/sentry/grouping/api.py:SecondaryGroupingConfigLoader.get_config_dict
            -> src/sentry/grouping/ingest/hashing.py:_calculate_secondary_hashes
  -> src/sentry/event_manager.py:handle_existing_grouphash [secondary.existing_grouphash found]
  -> src/sentry/grouping/ingest/seer.py:maybe_send_seer_for_new_model_training [secondary existing grouphash found]
       [same subtree as primary match above]
  -> src/sentry/grouping/ingest/seer.py:maybe_check_seer_for_matching_grouphash [no primary or secondary match]
       -> src/sentry/grouping/ingest/seer.py:should_call_seer_for_grouping
            -> Feature flag checks, platform checks, project option checks
       -> src/sentry/grouping/ingest/seer.py:get_seer_similar_issues
            -> src/sentry/grouping/ingest/seer.py:_build_seer_request
            -> src/sentry/seer/signed_seer_api.py:SeerViewerContext.__init__
            -> src/sentry/seer/similarity/similar_issues.py:get_similarity_data_from_seer
                 -> HTTP POST to Seer service /v0/issues/similar-issues
       -> src/sentry/grouping/ingest/seer.py:record_did_call_seer_metric
  -> src/sentry/event_manager.py:handle_existing_grouphash [Seer match found]
  -> src/sentry/event_manager.py:create_group_with_grouphashes [no match at all]
       [SEE SECTION 8]
  -> src/sentry/grouping/ingest/hashing.py:maybe_run_background_grouping
       -> src/sentry/options/rollout.py:in_random_rollout
       -> src/sentry/grouping/api.py:BackgroundGroupingConfigLoader.get_config_dict
       -> src/sentry/grouping/ingest/hashing.py:_calculate_background_grouping
  -> src/sentry/grouping/ingest/metrics.py:record_hash_calculation_metrics
       -> src/sentry/utils/metrics.py:metrics.incr [multiple]
  -> src/sentry/grouping/ingest/config.py:update_or_set_grouping_config_if_needed
       -> src/sentry/models/project.py:Project.get_option / update_option
       -> django.core.cache:cache.get / cache.set
       -> src/sentry/utils/locking/lock.py:Lock.acquire (Redis lock)
       -> src/sentry/utils/audit.py:create_system_audit_entry [if config updated]
```

---

## 6. Existing Group Handling: `handle_existing_grouphash()`

```
src/sentry/event_manager.py:handle_existing_grouphash
  -> src/sentry/models/group.py:Group.objects.get
  -> src/sentry/grouping/ingest/utils.py:is_non_error_type_group
  -> src/sentry/grouping/ingest/utils.py:add_group_id_to_grouphashes
       -> src/sentry/models/grouphash.py:GroupHash.objects.filter().exclude().update()
  -> src/sentry/event_manager.py:_get_group_processing_kwargs
       -> src/sentry/event_manager.py:_materialize_metadata_many
            -> src/sentry/event_manager.py:get_event_type (DYNAMIC registry)
            -> EventType.get_metadata [polymorphic]
            -> src/sentry/event_manager.py:materialize_metadata
       -> src/sentry/event_manager.py:get_event_type [separate call for group kwargs]
       -> src/sentry/event_manager.py:materialize_metadata [separate call merging metadata]
  -> src/sentry/event_manager.py:_process_existing_aggregate
       [SEE SECTION 7]
  -> src/sentry/workflow_engine/processors/detector.py:ensure_association_with_detector
       -> src/sentry/models/detector.py:Detector.objects.filter [for error groups]
       -> src/sentry/models/detectorgroup.py:DetectorGroup.objects.get_or_create
```

---

## 7. Process Existing Aggregate: `_process_existing_aggregate()`

```
src/sentry/event_manager.py:_process_existing_aggregate
  -> src/sentry/eventstore/models.py:Event.get_event_type [check != TransactionEvent.key]
  -> src/sentry/event_manager.py:_is_placeholder_title [check event.search_message]
  -> src/sentry/event_manager.py:_get_updated_group_title
       -> src/sentry/event_manager.py:_is_real_title
       -> src/sentry/event_manager.py:_is_placeholder_title
  -> src/sentry/event_manager.py:_handle_regression
       -> src/sentry/models/group.py:Group.is_resolved
       -> src/sentry/models/groupresolution.py:GroupResolution.has_resolution
       -> src/sentry/event_manager.py:has_pending_commit_resolution
            -> src/sentry/models/grouplink.py:GroupLink.objects.filter
            -> src/sentry/models/releasecommit.py:ReleaseCommit.objects.filter
            -> src/sentry/models/pullrequest.py:PullRequest.objects.filter
       -> src/sentry/event_manager.py:plugin_is_regression (DYNAMIC)
            -> src/sentry/plugins/base/manager.py:PluginManager.for_project
            -> Plugin.is_regression [for each enabled plugin]
       -> src/sentry/models/group.py:Group.objects.filter().exclude().update() [marks as UNRESOLVED]
       -> src/sentry/signals:issue_unresolved.send_robust [is_regression=True] (SIGNAL)
            -> src/sentry/receivers/features.py:record_issue_unresolved
                 -> src/sentry/models/featureadoption.py:FeatureAdoption.objects.record
            -> src/sentry/receivers/sentry_apps.py:send_issue_unresolved_webhook
                 -> Dispatches webhook to installed Sentry apps
       -> django.db.models.signals:post_save.send_robust [Group, is_regression and not enable-post-update-signal] (SIGNAL)
            -> src/sentry/issues/attributes.py:post_save_log_group_attributes_changed
       -> src/sentry/models/groupresolution.py:GroupResolution.objects.get [if is_regression and release]
       -> SQL DELETE sentry_groupresolution [if resolution found]
       -> src/sentry/models/activity.py:Activity.objects.filter [resolved_in_activity lookup]
       -> src/sentry/models/activity.py:Activity.update [update version data]
       -> src/sentry/models/release.py:follows_semver_versioning_scheme
       -> src/sentry/models/activity.py:Activity.objects.create_group_activity [SET_REGRESSION]
       -> src/sentry/models/grouphistory.py:record_group_history [REGRESSED]
       -> src/sentry/integrations/tasks/kick_off_status_syncs.py:kick_off_status_syncs.apply_async (CELERY)
            -> Queries GroupLink for external issues
            -> Dispatches sync_status_outbound.apply_async per external issue
       -> src/sentry/models/groupopenperiod.py:create_open_period [is_regression]
            -> src/sentry/models/groupopenperiod.py:GroupOpenPeriod.objects.create
            -> src/sentry/models/groupopenperiod.py:GroupOpenPeriodActivity.objects.create
  -> src/sentry/event_manager.py:_get_error_weighted_times_seen [if project in allowlist]
  -> src/sentry/tasks/process_buffer.py:buffer_incr [Group] (BACKEND)
       -> src/sentry/buffer/redis.py:RedisBuffer.incr
            -> Redis HINCRBY + ZADD operations
```

---

## 8. Group Creation: `create_group_with_grouphashes()`

```
src/sentry/event_manager.py:create_group_with_grouphashes
  -> src/sentry/grouping/ingest/utils.py:check_for_group_creation_load_shed
       -> src/sentry/killswitches.py:killswitch_matches_context
       -> raises HashDiscarded [if killswitch active]
  -> src/sentry/models/grouphash.py:GroupHash.objects.filter().select_for_update() [row lock]
  -> src/sentry/grouping/ingest/hashing.py:find_grouphash_with_group [double-check after lock]
  -> src/sentry/grouping/ingest/metrics.py:record_new_group_metrics [new group path]
       -> src/sentry/utils/metrics.py:metrics.incr [multiple]
  -> src/sentry/event_manager.py:_get_group_processing_kwargs [same as section 6]
  -> src/sentry/event_manager.py:_create_group
       -> src/sentry/event_manager.py:_get_next_short_id
            -> src/sentry/models/project.py:Project.next_short_id
                 -> src/sentry/models/counter.py:Counter.increment [SQL UPDATE + RETURNING]
       -> src/sentry/models/release.py:Release.objects.filter [verify first_release exists]
       -> src/sentry/event_manager.py:sdk_metadata_from_event
       -> src/sentry/event_manager.py:_get_severity_metadata_for_group
            [SEE SECTION 13 - SEVERITY]
       -> src/sentry/event_manager.py:_get_priority_for_group
       -> src/sentry/event_manager.py:_get_error_weighted_times_seen [if project in allowlist]
       -> src/sentry/models/group.py:Group.objects.create
       -> src/sentry/event_manager.py:_is_stuck_counter_error [on IntegrityError]
            -> checks psycopg2.errors.UniqueViolation
       -> src/sentry/event_manager.py:_handle_stuck_project_counter [if stuck counter detected]
            -> src/sentry/models/group.py:Group.objects.filter().aggregate(Max("short_id"))
            -> src/sentry/event_manager.py:_get_next_short_id [corrective]
       -> src/sentry/models/group.py:Group.objects.create [retry after unstuck]
       -> src/sentry/models/groupopenperiod.py:create_open_period
            -> src/sentry/models/groupopenperiod.py:GroupOpenPeriod.objects.create
            -> src/sentry/models/groupopenperiod.py:GroupOpenPeriodActivity.objects.create
  -> src/sentry/workflow_engine/processors/detector.py:associate_new_group_with_detector
       -> src/sentry/models/detector.py:Detector.objects.filter [for error groups]
       -> src/sentry/models/detectorgroup.py:DetectorGroup.objects.get_or_create
  -> src/sentry/grouping/ingest/utils.py:add_group_id_to_grouphashes
  -> src/sentry/event_manager.py:handle_existing_grouphash [race condition: lost to another process, uses locked grouphashes from select_for_update]
       [SEE SECTION 6]
```

---

## 9. Discard Event & Attachments

### `discard_event()`
```
src/sentry/event_manager.py:discard_event
  -> src/sentry/quotas/__init__.py:quotas.backend.refund (BACKEND)
       -> src/sentry/quotas/redis.py:RedisQuota.refund [default]
  -> src/sentry/utils/outcomes.py:track_outcome [FILTERED, per event]
       -> Kafka produce to outcomes topic
  -> src/sentry/utils/outcomes.py:track_outcome [FILTERED, per attachment]
  -> src/sentry/quotas/__init__.py:quotas.backend.refund [attachments] (BACKEND)
```

### `filter_attachments_for_group()`
```
src/sentry/event_manager.py:filter_attachments_for_group
  -> src/sentry/event_manager.py:get_max_crashreports(project, allow_none=True)
       -> model.get_option("sentry:store_crash_reports")
       -> src/sentry/lang/native/utils.py:convert_crashreport_count
  -> src/sentry/event_manager.py:get_max_crashreports(project.organization) [if first returned None]
       -> model.get_option("sentry:store_crash_reports")
       -> src/sentry/lang/native/utils.py:convert_crashreport_count
  -> src/sentry/models/eventattachment.py:get_crashreport_key
  -> src/sentry/event_manager.py:get_stored_crashreports
       -> django.core.cache:cache.get
       -> src/sentry/models/eventattachment.py:EventAttachment.objects.filter().count() [cache miss]
  -> src/sentry/event_manager.py:crashreports_exceeded [per attachment, checks limit]
  -> src/sentry/utils/outcomes.py:track_outcome [FILTERED, per exceeded crash report]
  -> django.core.cache:cache.set [update cached crash report count when exceeded]
  -> src/sentry/quotas/__init__.py:quotas.backend.refund [if refund needed] (BACKEND)
```

### `save_attachments()` / `save_attachment()`
```
src/sentry/event_manager.py:save_attachments
  -> src/sentry/event_manager.py:save_attachment [per attachment]
       -> src/sentry/attachments/__init__.py:CachedAttachment.load_data
       -> src/sentry/ratelimits/__init__.py:ratelimiter.backend.is_limited_with_value (BACKEND)
            [per-second and per-5-minute rate limits]
       -> src/sentry/utils/outcomes.py:track_outcome [RATE_LIMITED, if rate limited]
       -> src/sentry/models/eventattachment.py:EventAttachment.putfile
            -> File storage backend write
       -> src/sentry/models/eventattachment.py:EventAttachment.objects.create
       -> src/sentry/utils/outcomes.py:track_outcome [ACCEPTED]
       -> src/sentry/utils/outcomes.py:track_outcome [INVALID, on MissingAttachmentChunks]
```

---

## 10. TSDB Recording: `_tsdb_record_all_metrics()`

```
src/sentry/event_manager.py:_tsdb_record_all_metrics
  -> src/sentry/tsdb/__init__.py:tsdb.backend.incr_multi (BACKEND)
       Configured via settings.SENTRY_TSDB (default: "sentry.tsdb.dummy.DummyTSDB")
       Production: RedisSnubaTSDB routes per-model via model_backends dict:
       -> src/sentry/tsdb/redissnuba.py:RedisSnubaTSDB.incr_multi
            Routes based on TSDBModel:
            -> src/sentry/tsdb/redis.py:RedisTSDB.incr_multi [for group/project/release models]
                 -> Redis pipeline: HINCRBY on time-bucketed hash keys with TTL
            -> src/sentry/tsdb/dummy.py:DummyTSDB.incr_multi [for outcome-based models, write is no-op]
       -> src/sentry/tsdb/dummy.py:DummyTSDB.incr_multi [default config, all no-op]
  -> src/sentry/tsdb/__init__.py:tsdb.backend.record_multi (BACKEND)
       -> src/sentry/tsdb/redissnuba.py:RedisSnubaTSDB.record_multi
            -> src/sentry/tsdb/redis.py:RedisTSDB.record_multi
                 -> Redis PFADD (HyperLogLog for unique user counting)
       -> src/sentry/tsdb/dummy.py:DummyTSDB.record_multi [no-op]
  -> src/sentry/tsdb/__init__.py:tsdb.backend.record_frequency_multi (BACKEND)
       -> src/sentry/tsdb/redissnuba.py:RedisSnubaTSDB.record_frequency_multi
            -> src/sentry/tsdb/redis.py:RedisTSDB.record_frequency_multi
                 -> Redis ZINCRBY on sorted sets + Count-Min sketch estimation
       -> src/sentry/tsdb/dummy.py:DummyTSDB.record_frequency_multi [no-op]
```

---

## 11. Nodestore: `_nodestore_save_many()`

```
src/sentry/event_manager.py:_nodestore_save_many
  -> src/sentry/services/eventstore/processing.py:event_processing_store.get [for error events with groups] (BACKEND)
       Configured via settings.SENTRY_EVENT_PROCESSING_STORE
       -> src/sentry/services/eventstore/processing/redis.py:RedisClusterEventProcessingStore.get
            -> Redis Cluster KV GET (JSON deserialized)
       -> src/sentry/services/eventstore/processing/bigtable.py:BigtableEventProcessingStore.get [alt]
            -> Google Bigtable KV GET (JSON deserialized)
  -> src/sentry/utils/cache.py:cache_key_for_event [builds cache key for unprocessed event]
  -> src/sentry/usage_accountant/__init__.py:record [COGS tracking]
  -> src/sentry/db/models/fields/node.py:NodeData.save (via job["event"].data.save)
       -> src/sentry/services/nodestore/base.py:NodeStorage.set_subkeys
            -> src/sentry/services/nodestore/base.py:NodeStorage._encode [newline-separated JSON]
            -> src/sentry/services/nodestore/base.py:NodeStorage.set_bytes (BACKEND)
                 Configured via settings.SENTRY_NODESTORE (default: "sentry.services.nodestore.django.DjangoNodeStorage")
                 -> src/sentry/services/nodestore/django/backend.py:DjangoNodeStorage._set_bytes
                      -> Compresses data, then Node.objects.create_or_update (PostgreSQL)
                 -> src/sentry/services/nodestore/bigtable/backend.py:BigtableNodeStorage._set_bytes [production]
                      -> Compresses data (zlib/zstd), then BigtableKVStorage.set (Google Bigtable)
```

---

## 12. Eventstream: `_eventstream_insert_many()`

```
src/sentry/event_manager.py:_eventstream_insert_many
  -> src/sentry/features/__init__.py:features.has ["organizations:processing-error-analytics"] (DYNAMIC)
  -> random.random [1% sampling gate]
  -> src/sentry/analytics/__init__.py:analytics.record [EventProcessingErrorRecorded, 1% sample]
       -> src/sentry/analytics/event_manager.py:EventManager.record
  -> src/sentry/eventstore/models.py:Event.get_primary_hash [for non-transaction events]
  -> src/sentry/eventstream/__init__.py:eventstream.backend.insert (BACKEND)
       Configured via settings.SENTRY_EVENTSTREAM (default: "sentry.eventstream.snuba.SnubaEventStream")

       PATH A: src/sentry/eventstream/snuba.py:SnubaEventStream.insert
         -> src/sentry/eventstream/snuba.py:SnubaProtocolEventStream.insert
              -> src/sentry/quotas/__init__.py:quotas.backend.get_event_retention [retention days]
              -> src/sentry/eventstore/models.py:Event.get_raw_data
              -> Determines EventStreamEventType (Error/Transaction/Generic)
              -> src/sentry/eventstream/snuba.py:SnubaEventStream._send
                   -> HTTP POST to Snuba /tests/{entity}/eventstream
              -> src/sentry/eventstream/snuba.py:SnubaProtocolEventStream._forward_event_to_items [if EAP enabled]
                   -> HTTP POST to Snuba EAP_ITEMS_INSERT_ENDPOINT
         -> src/sentry/eventstream/base.py:EventStream._dispatch_post_process_group_task
              -> src/sentry/tasks/post_process.py:post_process_group.apply_async (CELERY)

       PATH B: src/sentry/eventstream/kafka/backend.py:KafkaEventStream.insert
         -> src/sentry/eventstream/snuba.py:SnubaProtocolEventStream.insert [_send resolves to KafkaEventStream._send override]
         -> src/sentry/eventstream/kafka/backend.py:KafkaEventStream._send
              -> Arroyo KafkaProducer.produce (KAFKA)
                   Topic determined by event type:
                   -> Topic.EVENTS [error events]
                   -> Topic.TRANSACTIONS [transaction events]
                   -> Topic.EVENTSTREAM_GENERIC [generic events]
         -> src/sentry/eventstream/kafka/backend.py:KafkaEventStream._send_item [if EAP enabled]
              -> Arroyo KafkaProducer.produce to Topic.SNUBA_ITEMS (KAFKA)
         [NOTE: KafkaEventStream.requires_post_process_forwarder() returns True]
         [post_process_group is dispatched by a separate Kafka consumer, NOT inline]
```

---

## 13. Severity Score: `_get_severity_metadata_for_group()`

```
src/sentry/event_manager.py:_get_severity_metadata_for_group
  -> src/sentry/receivers/rules.py:PLATFORMS_WITH_PRIORITY_ALERTS [import, platform allowlist]
  -> src/sentry/killswitches.py:killswitch_matches_context
  -> src/sentry/features/__init__.py:features.has ["organizations:seer-based-priority"] (DYNAMIC - handler chain)
  -> src/sentry/features/__init__.py:features.has ["projects:first-event-severity-calculation"]
  -> [returns {} if platform not in PLATFORMS_WITH_PRIORITY_ALERTS]
  -> [returns {} if group_type is not ErrorGroupType]
  -> src/sentry/utils/circuit_breaker.py:circuit_breaker_activated
       -> django.core.cache:cache.get [error count check]
  -> src/sentry/ratelimits/__init__.py:ratelimiter.backend.is_limited (BACKEND)
       [global rate limit check]
  -> src/sentry/ratelimits/__init__.py:ratelimiter.backend.is_limited (BACKEND)
       [per-project rate limit check]
  -> src/sentry/event_manager.py:_get_severity_score
       [returns (1.0, "log_level_fatal") early if FATAL]
       [returns (0.0, "log_level_info") early if INFO/DEBUG]
       -> src/sentry/event_manager.py:get_event_type (DYNAMIC registry)
       -> EventType.get_metadata [polymorphic]
       -> src/sentry/utils/safe.py:trim [truncate title to 128 chars]
       [returns (0.0, "bad_title") early if PLACEHOLDER_EVENT_TITLES]
       -> src/sentry/utils/event.py:has_stacktrace
       -> src/sentry/utils/event.py:is_handled
       -> src/sentry/seer/signed_seer_api.py:SeerViewerContext.__init__
       -> src/sentry/event_manager.py:make_severity_score_request
            -> src/sentry/seer/signed_seer_api.py:make_signed_seer_api_request
                 -> HTTP POST to Seer /v0/issues/severity-score
       -> orjson.loads [parse response]
       -> src/sentry/event_manager.py:update_severity_error_count [on error]
            -> django.core.cache:cache.incr / cache.set
       -> src/sentry/event_manager.py:update_severity_error_count(reset=True) [on success]
       -> sentry_sdk.capture_exception [on unknown error]
```

---

## 14. Dynamic Backend Summary

All backends resolved via `LazyServiceWrapper` + `import_string()` at first use:

| Module | Setting | Default Backend | Production Backend |
|--------|---------|----------------|-------------------|
| `eventstream` | `SENTRY_EVENTSTREAM` | `sentry.eventstream.snuba.SnubaEventStream` | `sentry.eventstream.kafka.KafkaEventStream` |
| `eventstore` (storage) | internal | `sentry.eventstore.snuba.SnubaEventStorage` | same |
| `tsdb` | `SENTRY_TSDB` | `sentry.tsdb.dummy.DummyTSDB` | `sentry.tsdb.redissnuba.RedisSnubaTSDB` |
| `nodestore` | `SENTRY_NODESTORE` | `sentry.services.nodestore.django.DjangoNodeStorage` | `sentry.services.nodestore.bigtable.BigtableNodeStorage` |
| `quotas` | `SENTRY_QUOTAS` | `sentry.quotas.Quota` | `sentry.quotas.redis.RedisQuota` |
| `buffer` | `SENTRY_BUFFER` | `sentry.buffer.Buffer` | `sentry.buffer.redis.RedisBuffer` |
| `ratelimiter` | `SENTRY_RATELIMITER` | `sentry.ratelimits.base.RateLimiter` | `sentry.ratelimits.redis.RedisRateLimiter` |
| `event_processing_store` | `SENTRY_EVENT_PROCESSING_STORE` | `sentry.services.eventstore.processing.redis.RedisClusterEventProcessingStore` | `BigtableEventProcessingStore` (GCP) |
| `cache` | `SENTRY_CACHE` | (required) | Redis-backed |

---

## 15. Complete Signal Dispatch Map

Signals sent during `EventManager.save()` and their receivers:

| Signal | Condition | Receivers |
|--------|-----------|-----------|
| `first_event_received` | Not raw, project has no first_event | `receivers/features.py:record_first_event`, `receivers/onboarding.py:record_first_event` |
| `first_event_with_minified_stack_trace_received` | Not raw, event has minified stack trace | `receivers/onboarding.py:record_event_with_first_minified_stack_trace_for_project` |
| `first_transaction_received` | Transaction path, not skip_send | `receivers/onboarding.py:record_first_transaction` |
| `first_insight_span_received` | Transaction path, per insight module | `receivers/onboarding.py:record_first_insight_span` |
| `issue_unresolved` | Regression detected | `receivers/features.py:record_issue_unresolved`, `receivers/sentry_apps.py:send_issue_unresolved_webhook` |
| `post_save` (Group) | Regression, not enable-post-update-signal | `issues/attributes.py:post_save_log_group_attributes_changed` |

---

## 16. Complete Celery Task Dispatch Map

Async tasks dispatched during `EventManager.save()`:

| Task | Condition | Location |
|------|-----------|----------|
| `post_process_group` | Always (via eventstream); inline for SnubaEventStream, via external forwarder for KafkaEventStream | `tasks/post_process.py` |
| `kick_off_status_syncs` | Regression detected | `integrations/tasks/kick_off_status_syncs.py` |
| `sync_status_outbound` | Per external issue (from kick_off_status_syncs) | `integrations/tasks/` |
| `schedule_invalidate_project_config` | Transaction path, new boosted release; also indirect via `ReleaseProject.post_save` signal | `tasks/relay.py` |
| `buffer_incr` (via buffer backend) | Group/GroupTombstone/ReleaseProject/RPE updates | `tasks/process_buffer.py` → `buffer/redis.py` |

---

## 17. Complete Kafka Production Map

Kafka messages produced during `EventManager.save()`:

| Topic | Condition | Producer Location |
|-------|-----------|-------------------|
| `Topic.EVENTS` | Error events (KafkaEventStream) | `eventstream/kafka/backend.py` |
| `Topic.TRANSACTIONS` | Transaction events (KafkaEventStream) | `eventstream/kafka/backend.py` |
| `Topic.EVENTSTREAM_GENERIC` | Generic events (KafkaEventStream) | `eventstream/kafka/backend.py` |
| `Topic.SNUBA_ITEMS` | EAP forwarding enabled | `eventstream/kafka/backend.py` |
| `Topic.INGEST_OCCURRENCES` | Performance problems detected (txn path) | `issues/producer.py` |
| Outcomes topic | Every event (accepted/filtered/rate_limited) | `utils/outcomes.py` |

---

## 18. Complete External HTTP Calls

| Destination | Condition | Caller |
|-------------|-----------|--------|
| Seer `/v0/issues/severity-score` | New error group, feature enabled | `event_manager.py:make_severity_score_request` |
| Seer `/v0/issues/similar-issues` | No grouphash match, Seer enabled | `grouping/ingest/seer.py:get_seer_similar_issues` |
| Seer `/v0/issues/similar-issues` | Existing match, training enabled | `seer/similarity/similar_issues.py:get_similarity_data_from_seer` |
| Snuba `/tests/{entity}/eventstream` | SnubaEventStream backend | `eventstream/snuba.py:SnubaEventStream._send` |
| Snuba `/api/v1/eap_insert` | EAP forwarding enabled | `eventstream/snuba.py:SnubaEventStream._forward_event_to_items` |

---

## 19. Branching Summary

The three top-level paths share these functions:

| Function | Error | Transaction | Generic |
|----------|-------|-------------|---------|
| `_get_or_create_release_many` | Y | Y | Y |
| `_get_event_user_many` | Y | Y | Y |
| `_derive_plugin_tags_many` | Y | Y | Y |
| `_derive_interface_tags_many` | Y | Y | Y |
| `_derive_client_error_sampling_rate` | Y | N | N |
| `_materialize_metadata_many` | N (via _get_group_processing_kwargs) | Y | Y |
| `_calculate_span_grouping` | N | Y | N |
| `_get_or_create_environment_many` | Y | Y | Y |
| `_get_or_create_group_environment_many` | Y | N | N |
| `_get_or_create_release_associated_models` | Y | Y | N |
| `_increment_release_associated_counts_many` | Y | N | N |
| `_get_or_create_group_release_many` | Y | N | N |
| `_tsdb_record_all_metrics` | Y | Y | N |
| `_materialize_event_metrics` | Y | Y | Y |
| `_nodestore_save_many` | Y | Y | Y |
| `_eventstream_insert_many` | Y | Y | N |
| `_track_outcome_accepted_many` | Y | Y | N |
| `assign_event_to_group` | Y | N | N |
| `_detect_performance_problems` | N | Y | N |
| `_send_occurrence_to_platform` | N | Y | N |
| `_record_transaction_info` | N | Y | N |
| `filter_attachments_for_group` | Y | N | N |
| `save_attachments` | Y | N | N |
