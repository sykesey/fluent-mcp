 **Context:** Flow API spec: Used to create a new Flow object, which specifies a Wofkflow composed of a definition, triggers, actions, logic and data.


```typescript
// spec to configure a Flow in Fluent using the Flow plugin
import { action, Flow, wfa, trigger } from '@servicenow/sdk/automation'

export const incidentSeverityAlertFlow = Flow(
    {
        $id: Now.ID['incident_severity_alert_flow'],
        name: 'Incident Severity Alert Flow',
        description: 'Notifies team based on incident severity and sets state to In Progress',
    },
    wfa.trigger(
        trigger.record.created,
        { $id: Now.ID['incident_created_trigger'] },
        { table: 'incident', condition: 'origin=NULL', run_flow_in: 'background' }
    ),
    (params) => {
        // Log the new incident
        wfa.action(
            action.core.log,
            { $id: Now.ID['log_incident'] },
            {
                log_level: 'info',
                log_message: `New incident: ${wfa.dataPill(params.trigger.current.short_description, 'string')}`,
            }
        )

        // High severity — notify manager + SMS all assignees
        wfa.flowLogic.if(
            {
                $id: Now.ID['check_high'],
                condition: `${wfa.dataPill(params.trigger.current.severity, 'string')}=1`,
            },
            () => {
                wfa.action(
                    action.core.sendNotification,
                    { $id: Now.ID['notify_manager'] },
                    {
                        table_name: 'incident',
                        record: wfa.dataPill(params.trigger.current.sys_id, 'reference'),
                        notification: 'high_severity_manager_notification',
                    }
                )

                wfa.flowLogic.forEach(
                    wfa.dataPill(params.trigger.current.additional_assignee_list, 'array.string'),
                    { $id: Now.ID['foreach_assignees'] },
                    () => {
                        wfa.action(
                            action.core.sendSms,
                            { $id: Now.ID['sms_assignee'] },
                            {
                                recipients: wfa.dataPill(params.trigger.current.additional_assignee_list, 'array.string'),
                                message: `High severity: ${wfa.dataPill(params.trigger.current.short_description, 'string')}`,
                            }
                        )
                    }
                )
            }
        )

        // Medium severity — SMS assignees only
        wfa.flowLogic.elseIf(
            {
                $id: Now.ID['check_medium'],
                condition: `${wfa.dataPill(params.trigger.current.severity, 'string')}=2`,
            },
            () => {
                wfa.flowLogic.forEach(
                    wfa.dataPill(params.trigger.current.additional_assignee_list, 'array.string'),
                    { $id: Now.ID['foreach_assignees_medium'] },
                    () => {
                        wfa.action(
                            action.core.sendSms,
                            { $id: Now.ID['sms_assignee_medium'] },
                            {
                                recipients: wfa.dataPill(params.trigger.current.additional_assignee_list, 'array.string'),
                                message: `Medium severity: ${wfa.dataPill(params.trigger.current.short_description, 'string')}`,
                            }
                        )
                    }
                )
            }
        )

        // Always set state to In Progress
        wfa.action(
            action.core.updateRecord,
            { $id: Now.ID['update_state'] },
            {
                table_name: 'incident',
                record: wfa.dataPill(params.trigger.current.sys_id, 'reference'),
                values: TemplateValue({ state: '2' }),
            }
        )
    }
)
```