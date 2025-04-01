function main(workbook: ExcelScript.Workbook, flowJson: string): string {
  // Parse the JSON data
  let flowDefinition: FlowDefinition;
  try {
    flowDefinition = JSON.parse(flowJson) as FlowDefinition;
  } catch (error) {
    throw new Error("Invalid JSON format provided.");
  }

  // Attempt to get the actions from multiple possible locations
  let actions: ActionsMap = flowDefinition.actions;
  if (
    !actions &&
    flowDefinition.body &&
    flowDefinition.body.properties &&
    flowDefinition.body.properties.definition &&
    flowDefinition.body.properties.definition.actions
  ) {
    actions = flowDefinition.body.properties.definition.actions;
  }

  if (!actions) {
    throw new Error("Invalid structure. Actions not found.");
  }

  // Initialize the Mermaid diagram with the enhanced configuration block
  let mermaidDiagram: { content: string } = {
    content: `---
config:
  layout: elk
---
graph TD
`,
  };

  // Set to track defined classes to prevent duplicates
  const definedClasses: Set<string> = new Set();

  // Set to track added connections to prevent duplicates
  const addedConnections: Set<string> = new Set();

  // Mapping from action name to its last nodes
  const actionLastNodesMap: { [actionName: string]: string[] } = {};

  // Global map of all actions with their full paths
  const globalActionsMap: { [key: string]: { action: Action; fullPath: string[] } } = {};

  // Function to generate unique and sanitized node identifiers based on action names
  function getNodeIdentifier(actionName: string): string {
    // Sanitize the actionName by replacing non-word characters with underscores
    const sanitizedActionName = actionName.replace(/\W/g, "_");

    // Ensure node ID starts with a letter
    const nodeId = /^[A-Za-z]/.test(sanitizedActionName)
      ? sanitizedActionName
      : `N_${sanitizedActionName}`;

    return nodeId;
  }

  // Function to sanitize labels for Mermaid syntax
  function sanitizeLabel(label: string): string {
    // Remove parentheses and other special characters that may interfere with Mermaid syntax
    return label.replace(/[()]/g, "").replace(/_/g, " ").trim() || "Unnamed Action";
  }

  // Object to keep track of processed actions to avoid infinite loops
  const processedActions: { [key: string]: boolean } = {};

  // Function to handle logging warnings
  function logWarning(message: string): void {
    console.log(`WARNING: ${message}`);
  }

  // Function to handle logging informational messages
  function logInfo(message: string): void {
    console.log(`INFO: ${message}`);
  }

  // Helper function to find actions that a given action depends on
  function getDependencies(action: Action): string[] {
    const runAfter: RunAfterMap = action.runAfter || {};
    return Object.keys(runAfter);
  }

  // Improved function to determine style based on action type
  function getStyleForActionType(actionType: string): { fill: string; stroke: string; shape: string; className: string } {
    const styleMap: { [key: string]: { fill: string; stroke: string; shape: string; className: string } } = {
      If: { fill: "#FFE0B2", stroke: "#FF6D00", shape: "diamond", className: "If" },
      Switch: { fill: "#BBDEFB", stroke: "#2962FF", shape: "diamond", className: "Switch" },
      Foreach: { fill: "#C8E6C9", stroke: "#00C853", shape: "rectangle", className: "Foreach" },
      Until: { fill: "#C8E6C9", stroke: "#00C853", shape: "rectangle", className: "Until" },
      SetVariable: { fill: "#FFF9C4", stroke: "#FFEB3B", shape: "rectangle", className: "SetVariable" },
      Scope: { fill: "#BBDEFB", stroke: "#2962FF", shape: "rectangle", className: "Scope" },
      Terminate: { fill: "#FFCDD2", stroke: "#C62828", shape: "rectangle", className: "Terminate" },
      RunScript: { fill: "#B3E5FC", stroke: "#0277BD", shape: "rectangle", className: "RunScript" },
      SendEmail: { fill: "#FFECB3", stroke: "#FFA000", shape: "rectangle", className: "SendEmail" },
      Compose: { fill: "#FFFFFF", stroke: "#000000", shape: "rectangle", className: "Compose" },
      Response: { fill: "#FFFFFF", stroke: "#000000", shape: "rectangle", className: "Response" },
      InitializeVariable: { fill: "#FFFFFF", stroke: "#000000", shape: "rectangle", className: "InitializeVariable" },
      AppendToStringVariable: { fill: "#FFF9C4", stroke: "#FFEB3B", shape: "rectangle", className: "SetVariable" },
      OpenApiConnection: { fill: "#FFFFFF", stroke: "#000000", shape: "rectangle", className: "OpenApiConnection" },
      Http: { fill: "#FFFFFF", stroke: "#000000", shape: "rectangle", className: "Http" },
      Expression: { fill: "#E1BEE7", stroke: "#8E24AA", shape: "rectangle", className: "Expression" },
      Workflow: { fill: "#FFCCBC", stroke: "#E64A19", shape: "rectangle", className: "Workflow" },
      // Add more styles as needed
    };

    return styleMap[actionType] || { fill: "#FFFFFF", stroke: "#000000", shape: "rectangle", className: "Default" };
  }

  // Function to build the global actions map
  function buildGlobalActionsMap(actions: ActionsMap, parentPath: string[] = []): void {
    for (const actionName in actions) {
      const action = actions[actionName];
      const fullPath = parentPath.concat(actionName);
      globalActionsMap[actionName] = { action, fullPath };

      // Recursively process child actions
      if (action.actions) {
        buildGlobalActionsMap(action.actions, fullPath);
      }

      // Process 'else' branch in 'If' actions
      if (action.type === "If" && action.else && action.else.actions) {
        buildGlobalActionsMap(action.else.actions, fullPath);
      }

      // Process cases in 'Switch' actions
      if (action.type === "Switch" && action.cases) {
        for (const caseName in action.cases) {
          const caseBranch = action.cases[caseName];
          if (caseBranch.actions) {
            buildGlobalActionsMap(caseBranch.actions, fullPath);
          }
        }
        // Process default case
        if (action.default && action.default.actions) {
          buildGlobalActionsMap(action.default.actions, fullPath);
        }
      }
    }
  }

  // Build the global actions map before processing
  buildGlobalActionsMap(actions);

  // Define the Start node first
  const startId: string = "Start";
  mermaidDiagram.content += `${startId}((Start))\n`;
  mermaidDiagram.content += `style ${startId} fill:#C8E6C9,stroke:#00C853,stroke-width:2px;\n`;

  // Helper function to get actions in order
  function getOrderedActionNames(actions: ActionsMap): string[] {
    return Object.keys(actions);
  }

  // Find actions with no dependencies (runAfter is empty)
  const actionNames: string[] = getOrderedActionNames(actions);
  const startingActions: string[] = actionNames.filter(actionName => getDependencies(actions[actionName]).length === 0);

  // Connect Start node to starting action nodes
  if (startingActions.length > 0) {
    for (let startActionName of startingActions) {
      const startActionId: string = getNodeIdentifier(startActionName);
      const connection: string = `${startId} --> ${startActionId}\n`;
      if (!addedConnections.has(connection)) {
        mermaidDiagram.content += connection;
        addedConnections.add(connection);
        logInfo(`Connected ${startId} to ${startActionId}`);
      }
    }
  } else {
    logWarning("No starting actions found to connect to Start node.");
  }

  /**
   * Main processing function.
   * Processes an action by:
   * 1. Processing its dependencies first.
   * 2. Defining the node (except for conditions and switches, which are defined inside subgraphs).
   * 3. Connecting from dependencies to this node.
   * 4. Processing child actions (if any).
   * 5. Storing the last nodes of this action.
   */
  function processAction(
    actionName: string,
    actions: ActionsMap,
    mermaidDiagram: { content: string },
    processedActions: { [key: string]: boolean },
    parentPath: string[] = []
  ): string[] {
    const action: Action = actions[actionName];

    if (!action) {
      logWarning(`Action "${actionName}" not found in actions map.`);
      return []; // Prevent node creation for non-existent actions
    }

    const nodeId: string = getNodeIdentifier(actionName);
    const nodeLabel: string = sanitizeLabel(actionName);

    // If already processed, return the last nodes
    if (processedActions[actionName]) {
      return actionLastNodesMap[actionName] || [nodeId];
    }
    processedActions[actionName] = true;

    // Process Dependencies First
    const dependencies: string[] = getDependencies(action);
    let dependencyLastNodes: string[] = [];

    dependencies.forEach((depName) => {
      // Find the dependency action in the global actions map
      if (!globalActionsMap.hasOwnProperty(depName)) {
        logWarning(`Dependency "${depName}" of action "${actionName}" not found in global actions map.`);
        return;
      }
      const depInfo = globalActionsMap[depName];
      const depAction = depInfo.action;

      // Check if dependency is valid (i.e., not creating a circular dependency)
      if (processedActions[depName]) {
        // Dependency already processed
      } else {
        // Recursively process dependencies
        const depLastNodes = processAction(
          depName,
          depAction.actions || actions,
          mermaidDiagram,
          processedActions,
          depInfo.fullPath
        );
        dependencyLastNodes = dependencyLastNodes.concat(depLastNodes);
      }

      // Connect from each last node of the dependency to this action
      const depNodeId: string = getNodeIdentifier(depName);
      const connection: string = `${depNodeId} --> ${nodeId}\n`;
      if (!addedConnections.has(connection)) {
        mermaidDiagram.content += connection;
        addedConnections.add(connection);
        logInfo(`Connected ${depNodeId} to ${nodeId}`);
      }
    });

    // Determine style based on action type
    const style = getStyleForActionType(action.type);
    const nodeShape = style.shape;
    const className = style.className;

    // Process different action types
    let lastNodes: string[] = [];
    switch (action.type) {
      case "If":
        // Create condition action with proper styling
        lastNodes = processConditionAction(actionName, action as ConditionAction, mermaidDiagram, processedActions, parentPath);
        break;
      case "Switch":
        // Create switch action with proper styling
        lastNodes = processSwitchAction(actionName, action as SwitchAction, mermaidDiagram, processedActions, parentPath);
        break;
      case "Foreach":
      case "Until":
      case "Scope":
        // Create scope-like actions with proper styling
        lastNodes = processScopeAction(actionName, action, mermaidDiagram, processedActions, parentPath);
        break;
      default:
        // Define the node with label and shape
        if (action.type === "Start") {
          mermaidDiagram.content += `${nodeId}((Start))\n`;
        } else {
          // Assign shapes based on action type
          if (nodeShape === "diamond") {
            mermaidDiagram.content += `${nodeId}{{${nodeLabel}}}\n`;
          } else if (nodeShape === "rectangle") {
            mermaidDiagram.content += `${nodeId}[${nodeLabel}]\n`;
          } else {
            // Default shape
            mermaidDiagram.content += `${nodeId}(${nodeLabel})\n`;
          }
        }

        // Apply class to the node
        mermaidDiagram.content += `class ${nodeId} ${className};\n`;

        // Define classDef if not already defined
        if (!definedClasses.has(className)) {
          mermaidDiagram.content += `classDef ${className} fill:${style.fill},stroke:${style.stroke},stroke-width:2px;\n`;
          definedClasses.add(className);
          logInfo(`Defined class ${className}`);
        }

        // Process child actions if any
        if (action.actions && Object.keys(action.actions).length > 0) {
          let childLastNodes: string[] = [];
          for (let childActionName in action.actions) {
            const childNodes: string[] = processAction(
              childActionName,
              action.actions,
              mermaidDiagram,
              processedActions,
              parentPath.concat(actionName)
            );
            const childId: string = getNodeIdentifier(childActionName);
            const connection: string = `${nodeId} --> ${childId}\n`;
            if (!addedConnections.has(connection)) {
              mermaidDiagram.content += connection;
              addedConnections.add(connection);
              logInfo(`Connected ${nodeId} to ${childId}`);
            }
            childLastNodes = childLastNodes.concat(childNodes);
          }
          lastNodes = childLastNodes.length > 0 ? childLastNodes : [nodeId];
        } else {
          // No child actions, last node is the current node
          lastNodes = [nodeId];
        }
        break;
    }

    // Store last nodes for this action
    actionLastNodesMap[actionName] = lastNodes;
    logInfo(`Action "${actionName}" has last nodes: ${lastNodes.join(", ")}`);

    return lastNodes;
  }

  /**
   * Enhanced function to process If actions and encapsulate them within a more visually appealing subgraph
   */
  function processConditionAction(
    actionName: string,
    action: ConditionAction,
    mermaidDiagram: { content: string },
    processedActions: { [key: string]: boolean },
    parentPath: string[]
  ): string[] {
    const nodeId: string = getNodeIdentifier(actionName);
    const nodeLabel: string = sanitizeLabel(actionName);

    // Start of subgraph for If Condition with improved styling
    mermaidDiagram.content += `subgraph "${nodeLabel} [If Condition]"\n`;
    logInfo(`Started subgraph for If Condition: ${nodeLabel}`);

    // Define the condition node inside the subgraph
    const style = getStyleForActionType(action.type);
    mermaidDiagram.content += `${nodeId}{{${nodeLabel}}}\n`;
    mermaidDiagram.content += `class ${nodeId} ${style.className};\n`;

    // Process 'ifTrue' branch
    let trueLastNodes: string[] = [];
    if (action.actions && Object.keys(action.actions).length > 0) {
      let isFirstTrueAction = true;
      for (let trueActionName in action.actions) {
        const trueId: string = getNodeIdentifier(trueActionName);
        if (isFirstTrueAction) {
          const connection: string = `${nodeId} -- Yes --> ${trueId}\n`;
          if (!addedConnections.has(connection)) {
            mermaidDiagram.content += connection;
            addedConnections.add(connection);
            logInfo(`Connected ${nodeId} to ${trueId} (Yes branch)`);
          }
          isFirstTrueAction = false;
        }
        const trueChildLastNodes: string[] = processAction(
          trueActionName,
          action.actions,
          mermaidDiagram,
          processedActions,
          parentPath.concat(actionName)
        );
        trueLastNodes = trueLastNodes.concat(trueChildLastNodes);
      }
    } else {
      trueLastNodes.push(nodeId);
      logWarning(`No actions found in 'ifTrue' branch of "${actionName}". Treating condition node as last node.`);
    }

    // Process 'ifFalse' branch
    let falseLastNodes: string[] = [];
    if (action.else && action.else.actions && Object.keys(action.else.actions).length > 0) {
      let isFirstFalseAction = true;
      for (let falseActionName in action.else.actions) {
        const falseId: string = getNodeIdentifier(falseActionName);
        if (isFirstFalseAction) {
          const connection: string = `${nodeId} -- No --> ${falseId}\n`;
          if (!addedConnections.has(connection)) {
            mermaidDiagram.content += connection;
            addedConnections.add(connection);
            logInfo(`Connected ${nodeId} to ${falseId} (No branch)`);
          }
          isFirstFalseAction = false;
        }
        const falseChildLastNodes: string[] = processAction(
          falseActionName,
          action.else.actions,
          mermaidDiagram,
          processedActions,
          parentPath.concat(actionName)
        );
        falseLastNodes = falseLastNodes.concat(falseChildLastNodes);
      }
    } else {
      falseLastNodes.push(nodeId);
      logWarning(`No actions found in 'ifFalse' branch of "${actionName}". Treating condition node as last node.`);
    }

    // End of subgraph
    mermaidDiagram.content += `end\n`;
    logInfo(`Ended subgraph for If Condition: ${nodeLabel}`);

    // The last nodes are the union of last nodes from both branches
    const lastNodes: string[] = trueLastNodes.concat(falseLastNodes);

    // Store last nodes for this condition action
    actionLastNodesMap[actionName] = lastNodes;
    logInfo(`Condition "${actionName}" has last nodes: ${lastNodes.join(", ")}`);

    return lastNodes;
  }

  /**
   * Enhanced function to process Switch actions and encapsulate them within a more visually appealing subgraph
   */
  function processSwitchAction(
    actionName: string,
    action: SwitchAction,
    mermaidDiagram: { content: string },
    processedActions: { [key: string]: boolean },
    parentPath: string[]
  ): string[] {
    const nodeId: string = getNodeIdentifier(actionName);
    const nodeLabel: string = sanitizeLabel(actionName);

    // Start of subgraph for Switch Condition with improved styling
    mermaidDiagram.content += `subgraph "${nodeLabel} [Switch Condition]"\n`;
    logInfo(`Started subgraph for Switch Condition: ${nodeLabel}`);

    // Define the switch node inside the subgraph
    const style = getStyleForActionType(action.type);
    mermaidDiagram.content += `${nodeId}{{${nodeLabel}}}\n`;
    mermaidDiagram.content += `class ${nodeId} ${style.className};\n`;

    let lastNodes: string[] = [];

    // Process Each Case
    if (action.cases && Object.keys(action.cases).length > 0) {
      for (let caseName in action.cases) {
        const caseBranch: CaseBranch = action.cases[caseName];
        if (caseBranch && caseBranch.actions && Object.keys(caseBranch.actions).length > 0) {
          let isFirstCaseAction = true;
          let caseLastNodes: string[] = [];
          for (let caseActionName in caseBranch.actions) {
            const caseActionId: string = getNodeIdentifier(caseActionName);
            if (isFirstCaseAction) {
              const connection: string = `${nodeId} -- ${sanitizeLabel(caseName)} --> ${caseActionId}\n`;
              if (!addedConnections.has(connection)) {
                mermaidDiagram.content += connection;
                addedConnections.add(connection);
                logInfo(`Connected ${nodeId} to ${caseActionId} (Case: ${caseName})`);
              }
              isFirstCaseAction = false;
            }
            const caseChildLastNodes: string[] = processAction(
              caseActionName,
              caseBranch.actions,
              mermaidDiagram,
              processedActions,
              parentPath.concat(actionName)
            );
            caseLastNodes = caseLastNodes.concat(caseChildLastNodes);
          }
          lastNodes = lastNodes.concat(caseLastNodes);
        } else {
          lastNodes.push(nodeId);
          logWarning(`No actions found in case "${caseName}" of "${actionName}". Treating condition node as last node.`);
        }
      }
    } else {
      logWarning(`No cases found in "${actionName}".`);
    }

    // Process Default Case
    if (action.default && action.default.actions && Object.keys(action.default.actions).length > 0) {
      for (let defaultActionName in action.default.actions) {
        const defaultActionId: string = getNodeIdentifier(defaultActionName);
        const connection: string = `${nodeId} -- Default --> ${defaultActionId}\n`;
        if (!addedConnections.has(connection)) {
          mermaidDiagram.content += connection;
          addedConnections.add(connection);
          logInfo(`Connected ${nodeId} to ${defaultActionId} (Default case)`);
        }
        const defaultChildLastNodes: string[] = processAction(
          defaultActionName,
          action.default.actions,
          mermaidDiagram,
          processedActions,
          parentPath.concat(actionName)
        );
        lastNodes = lastNodes.concat(defaultChildLastNodes);
      }
    }

    // End of subgraph
    mermaidDiagram.content += `end\n`;
    logInfo(`Ended subgraph for Switch Condition: ${nodeLabel}`);

    // Store last nodes for this switch action
    actionLastNodesMap[actionName] = lastNodes;
    logInfo(`Switch "${actionName}" has last nodes: ${lastNodes.join(", ")}`);

    return lastNodes;
  }

  /**
   * Enhanced function to process Scope-like actions with improved visual styling
   */
  function processScopeAction(
    actionName: string,
    action: Action,
    mermaidDiagram: { content: string },
    processedActions: { [key: string]: boolean },
    parentPath: string[]
  ): string[] {
    const nodeId: string = getNodeIdentifier(actionName);
    const nodeLabel: string = sanitizeLabel(actionName);

    // Start of subgraph for Scope with improved styling
    mermaidDiagram.content += `subgraph "${nodeLabel} [${action.type}]"\n`;
    logInfo(`Started subgraph for Scope: ${nodeLabel}`);

    // Define the scope node inside the subgraph
    const style = getStyleForActionType(action.type);
    mermaidDiagram.content += `${nodeId}[${nodeLabel}]\n`;
    mermaidDiagram.content += `class ${nodeId} ${style.className};\n`;

    let lastNodes: string[] = [];

    if (action.actions && Object.keys(action.actions).length > 0) {
      let isFirstAction = true;
      let childLastNodes: string[] = [];
      for (let childActionName in action.actions) {
        const childId: string = getNodeIdentifier(childActionName);
        if (isFirstAction) {
          const connection: string = `${nodeId} --> ${childId}\n`;
          if (!addedConnections.has(connection)) {
            mermaidDiagram.content += connection;
            addedConnections.add(connection);
            logInfo(`Connected ${nodeId} to ${childId} (Scope child)`);
          }
          isFirstAction = false;
        }
        const childNodes: string[] = processAction(
          childActionName,
          action.actions,
          mermaidDiagram,
          processedActions,
          parentPath.concat(actionName)
        );
        childLastNodes = childLastNodes.concat(childNodes);
      }
      lastNodes = childLastNodes.length > 0 ? childLastNodes : [nodeId];
    } else {
      // No child actions, last node is the current node
      lastNodes = [nodeId];
      logWarning(`No child actions found in Scope "${actionName}". Treating condition node as last node.`);
    }

    // End of subgraph
    mermaidDiagram.content += `end\n`;
    logInfo(`Ended subgraph for Scope: ${nodeLabel}`);

    // Store last nodes for this scope action
    actionLastNodesMap[actionName] = lastNodes;
    logInfo(`Scope "${actionName}" has last nodes: ${lastNodes.join(", ")}`);

    return lastNodes;
  }

  // Process each action
  for (let actionName of actionNames) {
    processAction(actionName, actions, mermaidDiagram, processedActions, []);
  }

  // Add default class definitions for all common action types
  if (!definedClasses.has('If')) {
    mermaidDiagram.content += `classDef If fill:#FFE0B2,stroke:#FF6D00,stroke-width:2px;\n`;
    definedClasses.add('If');
  }
  if (!definedClasses.has('Switch')) {
    mermaidDiagram.content += `classDef Switch fill:#BBDEFB,stroke:#2962FF,stroke-width:2px;\n`;
    definedClasses.add('Switch');
  }
  if (!definedClasses.has('Foreach')) {
    mermaidDiagram.content += `classDef Foreach fill:#C8E6C9,stroke:#00C853,stroke-width:2px;\n`;
    definedClasses.add('Foreach');
  }
  if (!definedClasses.has('Until')) {
    mermaidDiagram.content += `classDef Until fill:#C8E6C9,stroke:#00C853,stroke-width:2px;\n`;
    definedClasses.add('Until');
  }
  if (!definedClasses.has('SetVariable')) {
    mermaidDiagram.content += `classDef SetVariable fill:#FFF9C4,stroke:#FFEB3B,stroke-width:2px;\n`;
    definedClasses.add('SetVariable');
  }
  if (!definedClasses.has('Scope')) {
    mermaidDiagram.content += `classDef Scope fill:#BBDEFB,stroke:#2962FF,stroke-width:2px;\n`;
    definedClasses.add('Scope');
  }
  if (!definedClasses.has('Terminate')) {
    mermaidDiagram.content += `classDef Terminate fill:#FFCDD2,stroke:#C62828,stroke-width:2px;\n`;
    definedClasses.add('Terminate');
  }
  if (!definedClasses.has('InitializeVariable')) {
    mermaidDiagram.content += `classDef InitializeVariable fill:#FFFFFF,stroke:#000000,stroke-width:2px;\n`;
    definedClasses.add('InitializeVariable');
  }
  if (!definedClasses.has('OpenApiConnection')) {
    mermaidDiagram.content += `classDef OpenApiConnection fill:#FFFFFF,stroke:#000000,stroke-width:2px;\n`;
    definedClasses.add('OpenApiConnection');
  }
  if (!definedClasses.has('Expression')) {
    mermaidDiagram.content += `classDef Expression fill:#E1BEE7,stroke:#8E24AA,stroke-width:2px;\n`;
    definedClasses.add('Expression');
  }
  if (!definedClasses.has('Workflow')) {
    mermaidDiagram.content += `classDef Workflow fill:#FFCCBC,stroke:#E64A19,stroke-width:2px;\n`;
    definedClasses.add('Workflow');
  }

  // Return the diagram content
  return mermaidDiagram.content;
}

// Define interfaces for the JSON structure
interface FlowDefinition {
  actions?: ActionsMap;
  body?: {
    properties?: {
      definition?: {
        actions?: ActionsMap;
      };
    };
  };
}

interface ActionsMap {
  [key: string]: Action;
}

interface Action {
  type: string;
  runAfter?: RunAfterMap;
  actions?: ActionsMap; // For nested actions (e.g., in Scopes)
  else?: {
    actions?: ActionsMap;
  };
  cases?: {
    [key: string]: CaseBranch;
  };
  default?: {
    actions?: ActionsMap;
  };
}

interface RunAfterMap {
  [key: string]: string[];
}

interface ConditionAction extends Action {
  actions?: ActionsMap; // Actions in the 'ifTrue' branch
  else?: {
    actions?: ActionsMap; // Actions in the 'ifFalse' branch
  };
}

interface SwitchAction extends Action {
  cases?: {
    [key: string]: CaseBranch;
  };
  default?: {
    actions?: ActionsMap;
  };
}

interface ForeachAction extends Action {
  actions?: ActionsMap; // Actions inside the foreach loop
}

interface UntilAction extends Action {
  actions?: ActionsMap; // Actions inside the until loop
}

interface CaseBranch {
  actions: ActionsMap;
}
