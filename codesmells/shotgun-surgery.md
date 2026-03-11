Three concerns arise when desiging Shotgun Surgery—tight coupling and testability. They are:

**Shotgun Surgery:** Refers to a code smell where making a single change requires modifications in many places.

**Centralized Mediator Pattern:** Widely used in C# applications to decouple components.

**C# Example:**

```csharp
public class Mediator {
    private Dictionary<string, Action<object>> _actions = new Dictionary<string, Action<object>>();
    
    public void Register(string actionName, Action<object> action)
    {
        _actions[actionName] = action;
    }
    
    public void Execute(string actionName, object parameter)
    {
        if (_actions.ContainsKey(actionName))
        {
            _actions[actionName](parameter);
        }
    }
}
```

This eliminates Shotgun Surgery by centralizing logic. Benefits: easier maintenance, fewer errors, and improved scalability.

Advanced Topics:
- **Event Aggregation**: Add pub-sub events for cross-component communication.
- **Familywise Error Rate**: Adjust significance level when comparing multiple options.
- **Cross-cutting Concerns**: Handle logging, security, and caching centrally.

**How and Why**: Use Shotgun Surgery in C# to pinpoint areas needing refactoring. Address issues immediately for faster development cycles. Mastery unlocks ability to implement patterns like Mediator effectively.