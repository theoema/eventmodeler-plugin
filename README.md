# Event Modeler Claude Plugin

A Claude Code plugin that helps with event modeling - designing slices, writing scenarios, and creating read models with field mappings.

## Installation

```bash
# Add the marketplace
/plugin marketplace add theoema/eventmodeler-claude-plugin

# Install the plugin
/plugin install eventmodeler@eventmodeler
```

## Skills

### explore-eventmodel
Navigate and query event models using the `eventmodeler` CLI. Helps you:
- Get an overview of the model (slices, chapters, events, commands)
- Search for specific elements
- Check information completeness
- Understand slice structure and scenarios

### design-scenarios
Design Given-When-Then scenarios for slices. Helps you:
- Understand slice context by exploring nearby slices and events
- Design happy path, error, and edge case scenarios
- Add scenarios with realistic example field values and descriptions

### design-readmodel
Design read models with fields and mappings. Helps you:
- Propose fields based on use case
- Add fields with appropriate types
- Map fields from source events
- Verify completeness

## Requirements

This plugin works with the [Event Modeler](https://github.com/theoema/event-modeler) tool. You need the `eventmodeler` CLI available in your project.

## Usage

Once installed, the skills auto-invoke when relevant. You can also invoke them directly:

```
/eventmodeler:explore-eventmodel
/eventmodeler:design-scenarios
/eventmodeler:design-readmodel
```

Or just ask naturally:
- "Show me the slices in this model"
- "Help me add a scenario to the Place Order slice"
- "I want to design a read model for order history"

## Author

Theo Emanuelsson

## License

MIT
