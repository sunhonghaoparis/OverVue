<!--
Description:
  Displays Export Project button and allows users to export project
  Functionality includes: export prompts, creating folders and code, exporting images
  -->

<template>
  <q-btn class="nav-btn" color="secondary" label="Export">
    <q-menu class="dropdown" :offset="[0, 15]">
      <div class="column items-center">
        <p class="center">Export:</p>
        <q-btn class="menu-btn" no-caps color="secondary" label="Vue Project" @click="exportProject" />
        <q-btn class="menu-btn" id="export-component-nav-btn" no-caps color="secondary" label="Active Component"
          @click="useExportComponentBound" :disabled="!activeComponent.trim()" />
      </div>
    </q-menu>

  </q-btn>
</template>
<script>
import { useExportComponent } from "../composables/useExportComponent.js";
import { mapState } from "vuex";
const { fs, ipcRenderer } = window;

import writeNested from "../../mixins/writeNested";
import { result } from "lodash";


export default {
  name: "ExportProjectComponent",
  mixins: [writeNested],
  methods: {
    useExportComponentBound() {
      useExportComponent.bind(this)();
    },
    showExportProjectDialog() {
      ipcRenderer
        .invoke("exportProject", {
          title: "Choose location to save folder in",
          message: "Choose location to save folder in",
          nameFieldLabel: "Application Name",
        })
        .then((result) => {
          this.exportFile(result.filePath)
          alert('Successfully Exported')
        })
        .catch((err) => console.log(err));
    },
    exportProject: function () {

      this.showExportProjectDialog();
    },
    /**
     * @description creates the .js file
     * argument: location = path to dir
     * invokes: createRouterImports(this.componentMap['App'].children),
     *          createExport(this.componentMap['App'].children)
     *  */
    createRouter(location) {
      if (this.exportAsTypescript === "on") {
        fs.writeFileSync(
          path.join(location, "src", "router", "index.ts"),
          this.createRouterImports(this.routes) +
          this.createExport(this.routes)
        );
      } else {
        fs.writeFileSync(
          path.join(location, "src", "router", "index.js"),
          this.createRouterImports(this.routes) +
          this.createExport(this.routes)
        );
      }
    },
    /**
     * @description import routed components from the /views/ dir
     * @argument: this.componentMap['App'].children
     */
    createRouterImports(routes) {
      let str = "import { createRouter, createWebHistory } from 'vue-router';\n";
      for(let view in routes) {
          str += `import ${view} from '../views/${view}.vue';\n`;
      }
      return str;
    },
    /**
     * @description creates the `export default` code in <script>
     */
    createExport(routes) {
      let str = "export default createRouter({\n\thistory: createWebHistory(import.meta.env.BASE_URL),\n\troutes: [\n";
      for(let view in routes) {
          // HomeView route is initialized to "localhost:3000/" url
          if (view === "HomeView") {
            str += `\n\t\t{\n\t\t\tpath: '/',\n\t\t\tname:'${view}',\n\t\t\tcomponent:${view}\n\t\t},\n`;
          }
          // All other routes are initialized to "localhost:3000/<view Name>"
          else {
            str += `\n\t\t{\n\t\t\tpath: '/${view}',\n\t\t\tname:'${view}',\n\t\t\tcomponent:${view}\n\t\t},\n`;
          }
        //   str += `\n\t\t{\n\t\t\tpath: '/${appChildren[child].componentName}',\n\t\t\tname:'${appChildren[child].componentName}',\n\t\t\tcomponent:${appChildren[child].componentName}\n\t\t},\n`;
      }
      str += `\n\t\t]\n})`
      return str;
    },
    /**
     * @description: creates component code <template>, <script>, <style>
     * invokes writeTemplate, writeScript, writeStyle
     */

    writeRenderUnitTestString(componentName, htmlList) {

      const imports = `import { mount } from '@vue/test-utils'
import ${componentName} from '../../src/components/${componentName}.vue'
`

  const results = [imports];

  for (const el of htmlList) {
  const test = `
test('renders ${componentName}', () => {
  const wrapper = mount(${componentName})

  // customize your tests here; for more info please visit: https://github.com/vuejs/test-utils/
})`
    results.push(test);
  }

  return results.reduce((acc, ele) => acc += ele, '');
},

    createComponentTestCode(componentLocation, componentName, componentMap) {
      fs.writeFileSync(componentLocation, this.writeRenderUnitTestString(componentName, componentMap[componentName].htmlList))
    },

    createComponentCode(componentLocation, componentName, children, routes) {
      if (componentName === "App") {
        fs.writeFileSync(
          componentLocation + ".vue",
          this.writeTemplate(componentName, children, this.routes) +
          this.writeStyle(componentName)
        );
      } else {
        fs.writeFileSync(
          componentLocation + ".vue",
          this.writeComments(componentName) +
          this.writeTemplate(componentName, children, this.routes) +
          this.writeScript(componentName, children) +
          this.writeStyle(componentName)
        );
      }
    },
    // creates assets folder
    createAssetFile(targetLocation, assetLocation) {
      let saved = remote.nativeImage.createFromPath(assetLocation);
      let urlData = saved.toPNG();
      fs.writeFileSync(targetLocation + ".png", urlData);
    },
    writeTemplateTag(componentName) {
      // create reference object - replace later
      const htmlElementMap = {
        div: ["<div", "</div>"],
        button: ["<button", "</button>"],
        form: ["<form", "</form>"],
        img: ["<img", ""], //single
        link: ['<a href="#"', ""], //single
        list: ["<li", "</li>"],
        paragraph: ["<p", "</p>"],
        "list-ol": ["<ol", "</ol>"],
        "list-ul": ["<ul", "</ul>"],
        input: ["<input", ""], //single
        navbar: ["<nav", "</nav>"],
        header: ["<header", "</header>"],
        footer: ["<footer", "</footer>"],
        meta: ["<meta", "</meta>"],
        h1: ["<h1", "</h1>"],
        h2: ["<h2", "</h2>"],
        h3: ["<h3", "</h3>"],
        h4: ["<h4", "</h4>"],
        h5: ["<h5", "</h5>"],
        h6: ["<h6", "</h6>"],
        'e-button':[`<el-button type="info"`,`</el-button>`],
          'e-input':["<el-input", "</el-input>"],
          'e-link': [`<el-link type="primary">primary</el-link>
          <el-link type="success">success</el-link>
          <el-link type="info">info</el-link>
          <el-link type="warning">warning</el-link>
          <el-link type="danger"`, `danger</el-link>`],
          'e-form': ["<el-form", "</el-form>"],
          'e-checkbox': ["<el-checkbox", "</el-checkbox>"],
          'e-checkbox-button': ["<el-checkbox-button", "</el-checkbox-button>"],
          'e-date-picker': ["<el-date-picker", "</el-date-picker>"],
          'e-slider':["<el-slider", "</el-slider>"],
          'e-card': ["<el-card", "</el-card>"],
          'e-alert': [`<el-alert title="success alert" type="success"></el-alert>
          <el-alert title="info alert" type="info"></el-alert>
          <el-alert title="warning alert" type="warning"></el-alert>
          <el-alert title="danger alert" type="danger"`, `</el-alert>`],
          'e-dropdown': [ `<el-dropdown split-button type="primary" @click="handleClick">
          Dropdown List
          <template #dropdown>
           <el-dropdown-menu>
            <el-dropdown-item>
            Action 1
          </el-dropdown-item>
          <el-dropdown-item>
          Action 2
        </el-dropdown-item>
          </el-dropdown-menu>
          </template`, `
          </el-dropdown>`],
          'e-tag': [`<el-tag>Tag 1</el-tag>
     <el-tag class="ml-2" type="success">Tag 2</el-tag>
     <el-tag class="ml-2" type="info">Tag 3</el-tag>
     <el-tag class="ml-2" type="warning">Tag 4</el-tag>
     <el-tag class="ml-2" type="danger"`, `Tag 5</el-tag>`],

     'e-badge': [`<el-badge :value="12" class="item">
     <el-button>comments</el-button>
   </el-badge>
   <el-badge :value="3" class="item">
     <el-button>replies</el-button>
   </el-badge>
   <el-badge :value="1" class="item" type="primary">
     <el-button>comments</el-button>
   </el-badge>
   <el-badge :value="2" class="item" type="warning">
     <el-button>replies</el-button`,
     `
     </el-badge>`],
      };

      // iterate through component's htmllist
      let htmlArr = this.componentMap[componentName].htmlList;
      let outputStr = ``;
      // eslint-disable-next-line no-unused-vars
      for (let el of htmlArr) {
          if (!el.text) {
            outputStr += `    <${el}/>\n`;
           } else {
            outputStr += `    `;
            outputStr += htmlElementMap[el.text][0];
          //if conditional to check class
          if (el.class !== "") {
            outputStr += " " + "class = " + `"${el.class}"`;
          }
          if (el.binding !== "") {
            outputStr += " " + "v-model = " + `"${el.binding}"`;
          }
          outputStr += ">";
          if (el.children.length) {
            outputStr += "\n";
            outputStr += writeNested(el.children, `    `);
            outputStr += `    `;
            outputStr += htmlElementMap[el.text][1];
            outputStr += `  \n`;
          } else {
            outputStr += htmlElementMap[el.text][1] + "\n";
          }
        }
      }
      return outputStr;
    },
    writeComments(componentName) {
      if (this.componentMap[componentName]?.noteList?.length > 0) {
        let commentStr = '<!--'
        this.componentMap[componentName].noteList.forEach((el) => {
          commentStr += "\n"
          commentStr += el;
        })
        commentStr += '\n-->\n\n'
        return commentStr;
      } else {
        return ''
      }
    },
    /**
     * @description creates the <router-link> boilerplate for /views/components
     * also creates the <template></template> tag for each component
     */
    writeTemplate(componentName, children, routes) {
      let str = "";
      let routeStr = "";

      if (componentName === "App") {
        str += `<div id="app">\n\t\t<div id="nav">\n`;

        for (let route in routes) {
          if (route === "HomeView") {
            str += `\t\t\t<router-link to="/" class = "componentLinks">${route}</router-link>\n`;
          }
          else {
            str += `\t\t\t<router-link to="/${route}" class = "componentLinks">${route}</router-link>\n`;
          }
        }
          str += `\t\t</div>\n\t\t<router-view class = "router-view"></router-view>\n`;
        }

      // writes the HTML tag boilerplate
      let templateTagStr = this.writeTemplateTag(componentName);

      // Add import component string to routes template
      if (this.routes.hasOwnProperty(componentName)){
        const arrOfChildComp = this.componentMap[componentName].children;
        arrOfChildComp.forEach(childName => {
          let childNameClass = this.componentMap[childName].htmlAttributes.class;
          let childNameClassFullStr = (childNameClass === "") ? "" : ` class = '${childNameClass}'`;
          routeStr += `<${childName}${childNameClassFullStr}></${childName}>\n`
        });

        return `<template>\n  <div id = "${componentName}">\n${templateTagStr}${routeStr}\t</div>\n</template>`;
      }

      //adds class/id into code snippet with exporting
      if (this.componentMap[componentName].htmlAttributes) {

        let compID = this.componentMap[componentName].htmlAttributes.id;
        let compClass = (this.routes.hasOwnProperty(componentName)) ? componentName : this.componentMap[componentName].htmlAttributes.class;

        if (compClass !== "" && compID !== "") {

          return `<template>\n  <div id = "${compID}" class = "${compClass}">\n${templateTagStr}${routeStr}  \n\t</div>\n</template>`;
        }
        else if (compClass !== "" && compID === "") {

          return `<template>\n  <div class = "${compClass}">\n${templateTagStr}${routeStr}  \n\t</div>\n</template>`;
        }
        else if (compClass === "" && compID !== "") {

          return `<template>\n  <div id = "${compID}">\n${templateTagStr}${routeStr}  </div>\n</template>`;
        }
        else {

          return `<template>\n  <div>\n\t${str}${templateTagStr}${routeStr}  </div>\n</template>`;
        }
      }
      else {
        return `<template>\n\t${str}${templateTagStr}${routeStr}\t</div>\n</template>`
      }
    },
    /**
     * @description imports child components into <script>
     */
    writeScript(componentName, children) {
      // add import mapstate and mapactions if they exist
      const currentComponent = this.componentMap[componentName];
      const routes = Object.keys(this.routes);

      // Writes script boilerplate for non-route components
      if (!routes.includes(componentName)) {
        let imports = "";
        if (currentComponent.actions.length || currentComponent.state.length) {
          imports += "import { ";
          if (currentComponent.actions.length && currentComponent.state.length) {
            imports += "mapState, mapActions";
          }
          else if (currentComponent.state.length) {
            imports += "mapState";
          }
          else {
            imports += "mapActions";
          }
          imports += ' } from "vuex";\n';
        }
        // if in Typescript mode, import defineComponent
        if (this.exportAsTypescript === "on") {
          imports += 'import { defineComponent } from "vue";\n';
        }

        let childrenComponentNames = "";

        let data = "";
        data += "  data () {\n    return {";
        if (currentComponent.props.length) {
          currentComponent.props.forEach((prop) => {
            data += `\n      ${prop}: "PLACEHOLDER FOR VALUE",`;
          });
        }
          this.routes.HomeView.forEach((element) => {
            element.htmlList.forEach((html) => {
              if(html.binding !== '') {
                data += `\n      ${html.binding}: "PLACEHOLDER FOR VALUE",`;
              }
            })
          })
          data += "\n";
          data += "    }\n";
          data += "  },\n";

        // if true add computed section and populate with state
        let computed = "";
        if (currentComponent.state.length) {
          computed += "  computed: {";
          computed += "\n    ...mapState([";
          currentComponent.state.forEach((state) => {
            computed += `\n      "${state}",`;
          });
          computed += "\n    ]),\n";
          computed += "  },\n";
        }
        // if true add methods section and populate with actions
        let methods = "";
        if (currentComponent.actions.length) {
          methods += "  methods: {";
          methods += "\n    ...mapActions([";
          currentComponent.actions.forEach((action) => {
            methods += `\n      "${action}",`;
          });
          methods += "\n    ]),\n";
          methods += "  },\n";
        }
        // concat all code within script tags
        let output;
        if (this.exportAsTypescript === "on") {
          output = "\n\n<script lang='ts'>\n";

          output += imports + "\nexport default defineComponent ({\n  name: '" + componentName + "'";
        } else {
          output = "\n\n<script>\n";

          output += imports + "\nexport default {\n  name: '" + componentName + "'";

        }
        output += ",\n  components: {\n";
     
        output += childrenComponentNames + "  },\n";
        output += data;
        output += computed;
        output += methods;
        // eslint-disable-next-line no-useless-escape
        if (this.exportAsTypescript === "on") {
          output += "});\n<\/script>";
        } else {
          output += "};\n<\/script>";
        }
        return output;
      }
      // Write script for route components.
      else {
        let str = "";
        let childrenComponentNames = "";
        let childComponentImportNames = "";
        const arrOfChildComp = this.componentMap[componentName].children;

        if (componentName !== "App"){
          arrOfChildComp.forEach(childName => {
            // Build child component text string
            if (childName !== arrOfChildComp[arrOfChildComp.length - 1]){
              childrenComponentNames += "    " + childName + ",\n";
            }
            else {
              childrenComponentNames += "    " + childName + "\n";
            }

            // Build child component import text string
            childComponentImportNames += `import ${childName} from '../components/${childName}.vue';\n`
          })
        }

        // eslint-disable-next-line no-useless-escape
        if (this.exportAsTypescript === "on") {
          return `\n\n<script lang="ts">\nimport { defineComponent } from "vue";\n ${str}\nexport default defineComponent ({\n  name: '${componentName}',\n  components: {\n${childrenComponentNames}  }\n});\n<\/script>`;
        }
        str += "\n\n<script>";
        str += `\n${childComponentImportNames}`;
        str += `\n\nexport default {`
        str += `\n  components: {`
        str += `\n${childrenComponentNames}  }\n};`;
        str += `\n<\/script>`;
        return str
      }
    },
    writeStyle(componentName) {
      let htmlArray = this.componentMap[componentName].htmlList;
      let styleString = "";
      // Add grid css property to view component div
      // adds view component id grid style and adds child component css styling
      if (this.routes.hasOwnProperty(componentName)) {
        styleString += `#${componentName} {\n\tdisplay: grid; \n\tgrid-template-columns: repeat(${this.gridLayout[0]}, 1fr);\n\tgrid-template-rows: repeat(${this.gridLayout[1]}, 1fr);\n\tgrid-column-gap: 0px;\n\tgrid-row-gap: 0px;\n}\n`;
        this.routes[componentName].forEach((element) => {
          let styleSelector = (element.htmlAttributes.class === "") ? element.htmlList[0].text : '.' + element.htmlAttributes.class;
          styleString += `${styleSelector} {\n\tbackground-color: ${element.color};
\tgrid-area: ${element.htmlAttributes.gridArea[0]} / ${element.htmlAttributes.gridArea[1]} / ${element.htmlAttributes.gridArea[2]} / ${element.htmlAttributes.gridArea[3]};
\tz-index: ${element.z};
}\n`
        });
      };

    // Add default styling to App
    if (componentName === "App") {
      return `\n\n<style scoped>\n#nav {
    margin: auto;
    text-align: center;
    display: flex;
    justify-content: space-between;
    padding: 1rem 2rem;
    background: #cfd8dc;
    border: 1px solid black;
	  width:50%;
}
.router-view {
  margin:auto;
  background-color: gray;
  height: 720px;
  width: 1280px;
}
</style >`
    } else return `\n\n<style scoped>\n${styleString}</style >`;
    },
    // create Firebase config for OAuth
    createFirebaseConfigFile(location) {
      if(this.$store.state.exportOauth ==='on'){
        let str = `import { initializeApp } from 'firebase/app';`;
      str += `\n\tconst firebaseConfig = {`;
      str += `\n\tapiKey: "AIzaSyBR4o9xj4LtDaZ37-mC-FqRQWaz67_9Fq0",`;
      str += `\n\tauthDomain: "oauth-74279.firebaseapp.com",`;
      str += `\n\tprojectId: "oauth-74279",`;
      str += `\n\tstorageBucket: "oauth-74279.appspot.com",`;
      str += `\n\tmessagingSenderId: "91801023441",`;
      str += `\n\tappId: "1:91801023441:web:4d923f26f5ce9c7384e6f0",`;
      str += `\n\tmeasurementId: "G-ZZQMS6RRWR"`;
      str += `\n};`;
      str += `\nconst firebaseApp = initializeApp(firebaseConfig);`;
      str += `\nexport default firebaseApp`;

      fs.writeFileSync(path.join(location, "firebaseConfig.js"), str);
      }
    },

    createjestConfigFile(location){
      if(this.$store.state.importTest ==='on'){
      let str = `module.exports = {`;
        str += `\n\tpreset: '@vue/cli-plugin-unit-jest'`;
        str += `\n}`
      fs.writeFileSync(path.join(location,"jest.config.js"), str);
      }
    },
    createbabelConfigFile(location){
      if(this.$store.state.importTest ==='on'){
      let str = `module.exports = {`;
        str += `\n\tpresets: [`;
        str += `\n\t\t'@vue/cli-plugin-babel/preset'`;
        str += `\n\t]`;
        str += `\n}`
      fs.writeFileSync(path.join(location,"babel.config.js"), str);
      }
    },
    createOauthFile(location){
      if(this.$store.state.exportOauth ==='on'||this.$store.state.exportOauthGithub ==='on'){
      let str = `<template>`;
      str += `\n\t<!-- you can see the username when you log in -->`;
      str += `\n\t<h1 v-if="user">Username: {{ user }}</h1>`;
      str += `\n\t<div id="logout" v-if="isSignedIn">`;
      str += `\n\t\t<button @click="handleSignOut">logout</button>`;
      str += `\n\t</div>`;
      if(this.$store.state.exportOauth ==='on'){
      str += `\n\t<div id="GoogleSignIn" v-if="!isSignedIn">`;
      str += `\n\t\t<h3>Google Signin</h3>`;
      str += `\n\t\t<button @click="handleSignInGoogle">login</button>`;
      str += `\n\t</div>`;
      }

      if(this.$store.state.exportOauthGithub ==='on'){
      str += `\n\t<div id="GitHubSignIn" v-if="!isSignedIn">`;
      str += `\n\t\t<h3>GitHub Signin</h3>`;
      str += `\n\t\t<button @click="handleSignInGitHub">login</button>`;
      str += `\n\t</div>`;
      }

      str += `\n</template>`;
      str += `\n\n<script>`;
      str += `\nimport firebaseConfig from '../../firebaseConfig';`;
      str += `\nimport { getAuth, signInWithPopup, signOut, GoogleAuthProvider, TwitterAuthProvider, GithubAuthProvider } from "firebase/auth";`;
      str += `\nfirebaseConfig`;
      if(this.$store.state.exportOauth ==='on'){
      str += `\nconst provider = new GoogleAuthProvider();`;
      }
      if(this.$store.state.exportOauthGithub ==='on'){
        str += `\nconst providerGithub = new GithubAuthProvider();`;
      }
      str += `\nconst auth = getAuth();`;
      str += `\n\nexport default {`;
      str += `\n\tname: 'OauthComponent',`;
      str += `\n\tprops: {`;
      str += `\n\t},`;
      str += `\n\tdata() {`;
      str += `\n\t\treturn {`;
      str += `\n\t\t\tuser: '',`;
      str += `\n\t\t\tisSignedIn: false,`;
      str += `\n\t\t}`;
      str += `\n\t },`;
      str += `\n\tmethods: {`;
        if(this.$store.state.exportOauth ==='on'){
      str += `\n\t\thandleSignInGoogle() {`;
      str += `\n\t\t\tsignInWithPopup(auth, provider)`;
      str += `\n\t\t\t\t.then((result) => { `;
      str += `\n\t\t\t\t\tthis.user = result.user.displayName;`;
      str += `\n\t\t\t\t\tthis.isSignedIn = true;`;
      str += `\n\t\t\t\t}).catch((error) => {`;
      str += `\n\t\t\t\t\tconsole.log(error);`;
      str += `\n\t\t\t\t});`;
      str += `\n\t\t},`;
        }
        if(this.$store.state.exportOauthGithub ==='on'){
      str += `\n\t\thandleSignInGitHub() {`;
      str += `\n\t\t\tsignInWithPopup(auth, providerGithub)`;
      str += `\n\t\t\t\t.then((result) => { `;
      str += `\n\t\t\t\t\tthis.user = result.user.displayName;`;
      str += `\n\t\t\t\t\tthis.isSignedIn = true;`;
      str += `\n\t\t\t\t}).catch((error) => {`;
      str += `\n\t\t\t\t\tconsole.log(error);`;
      str += `\n\t\t\t\t});`;
      str += `\n\t\t},`;
        }

      str += `\n\t\thandleSignOut() {`;
      str += `\n\t\t\tsignOut(auth).then(() => {`;
      str += `\n\t\t\t\t\tthis.user = ''; `;
      str += `\n\t\t\t\t\tthis.isSignedIn = false;`;
      str += `\n\t\t\t\t}).catch((error) => {`;
      str += `\n\t\t\t\t\tconsole.log(error);`;
      str += `\n\t\t\t\t});`;
      str += `\n\t\t}`;
      str += `\n\t}`;
      str += `\n}`;
      str += `\n<\/script>`;
      str += `\n<style scoped>`;
        str += `\n</style>`;
      fs.writeFileSync(path.join(location,"src","components","oauth.vue"), str);
      }
    },
    createIndexFile(location) {
      let str = `<!DOCTYPE html>\n<html lang="en">\n\n<head>`;
      str += `\n\t<meta charset="utf-8">`;
      str += `\n\t<meta http-equiv="X-UA-Compatible" content="IE=edge">`;
      str += `\n\t<meta name="viewport" content="width=device-width,initial-scale=1.0">`;
      str += `\n\t<link rel="icon" href="/favicon.ico">`;
      str += `\n\t<title>New OverVue Project</title>`;
      str += `\n</head>\n\n`;
      str += `<body>`;
      str += `\n\t<noscript>`;
      str += `\n\t\t<strong>We're sorry but vue-boiler-plate-routes doesn't work properly without JavaScript enabled. Please enable it
      to continue.</strong>`;
      str += `\n\t</noscript>`;
      str += `\n\t<div id="app"></div>`;
      if (this.exportAsTypescript === "on") {
        str += `\n\t<script type="module" src="/src/main.ts"><\/script>`;
      } else {
        str += `\n\t<script type="module" src="/src/main.js"><\/script>`;
      }
      str += `\n</body>\n\n`;
      str += `</html>\n`;
      fs.writeFileSync(path.join(location, "index.html"), str);
    },
    // creates main.js boilerplate
    createMainFile(location) {
      let str = `import { createApp } from 'vue';`;
      str += `\nimport store from './store'`
      str += `\nimport App from './App.vue';`;
      str += `\nimport router from './router';\n`;
      str+= `\nimport ElementPlus from 'element-plus';`
      str+=`\nimport 'element-plus/dist/index.css';`
      str += `\nconst app = createApp(App);`;
      str += `\napp.use(router);`;
      str += `\napp.use(store)`;
      str+=`\napp.use(ElementPlus);`;
      str += `\napp.mount('#app');`;

      // if using typescript, export with .ts extension
      if (this.exportAsTypescript === "on") {
        fs.writeFileSync(path.join(location, "src", "main.ts"), str);
      } else {
        fs.writeFileSync(path.join(location, "src", "main.js"), str);
      }
    },
    createViteConfig(location) {
      let str = `import { fileURLToPath, URL } from 'url';\n\n`;
      str += `import { defineConfig } from 'vite';\n`;
      str += `import vue from '@vitejs/plugin-vue';\n\n`;
      str += `export default defineConfig({\n`
      str += `\tplugins: [vue()],\n`
      str += `\tresolve: {\n`
      str += `\t\t alias: {\n`
      str += `\t\t\t'@': fileURLToPath(new URL('./src', import.meta.url))\n`
      str += `\t\t}\n\t}\n})`

      // if using typescript, export with .ts extension
      if (this.exportAsTypescript === "on") {
        fs.writeFileSync(path.join(location, "vite.config.ts"), str);
      } else {
        fs.writeFileSync(path.join(location, "vite.config.js"), str);
      }
    },
    createESLintRC(location) {
      let str;
      if (this.exportAsTypescript === "on") {
        str += `require("@rushstack/eslint-patch/modern-module-resolution");\n\n`;
      }
      str += `module.exports = {\n`;
      str += `\t"root": true,\n`;
      str += `\t"extends": [\n`;
      str += `\t\t"plugin:vue/vue3-essential",\n`
      str += `\t\t"eslint:recommended"`
      if (this.exportAsTypescript === "on") {
        str += `,\n\t\t"@vue/eslint-config-typescript/recommended"\n`
      }
      str += `\n\t],\n`
      str += `\t"env": {\n`
      str += `\t\t"vue/setup-compiler-macros": true\n`
      str += `\t}\n}`
      fs.writeFileSync(path.join(location, ".eslintrc.cjs"), str);
    },
    createTSConfig(location) {
      if (this.exportAsTypescript === "on") {
        let str = `{\n\t"extends": "@vue/tsconfig/tsconfig.web.json",\n\t"include": ["env.d.ts", "src/**/*", "src/**/*.vue"],\n\t"compilerOptions": {\n\t\t"baseUrl": ".",\n\t\t"paths": {\n\t\t\t"@/*": ["./src/*"]\n\t\t}\n\t},`;
        str += `\t"references": [\n`
        str += `\t\t{\n\t\t\t"path": "./tsconfig.vite-config.json"\n\t\t}\n\t]\n}`
        fs.writeFileSync(path.join(location, "tsconfig.json"), str);
      } else {
        return;
      }
    },
    createTSViteConfig(location) {
      if (this.exportAsTypescript === "on") {
        let str = `{\n\t"extends": "@vue/tsconfig/tsconfig.node.json",\n\t"include": ["vite.config.*"],\n\t"compilerOptions": {\n\t\t"composite": true,\n\t\t"types": ["node", "viteset"]\n\t}\n}`;
        fs.writeFileSync(path.join(location, "tsconfig.vite-config.json"), str);
      } else {
        return;
      }
    },
    createTSDeclaration(location) {
      if (this.exportAsTypescript === "on") {
        let str = `/// <reference types="vite/client" />`;
        fs.writeFileSync(path.join(location, "env.d.ts"), str);
      } else {
        return;
      }
    },
    createStore(location) {
      let str = `import { createStore } from 'vuex';\n`;
      str += `\nconst store = createStore({`;
      str += `\n\tstate () {`;
      str += `\n\t\treturn {`;
      if (!this.userState.length) {
        str += `\n\t\t\t//placeholder for state`
      }
      for (let i = 0; i < this.userState.length; i++) {
        str += `\n\t\t\t${this.userState[i]}: "PLACEHOLDER FOR VALUE",`
        if (i === this.userState.length - 1) { str = str.slice(0, -1) }
      }
      str += `\n\t\t}`;
      str += `\n\t},`;
      str += `\n\tmutations: {`;
      if (!this.userActions.length) {
        str += `\n\t\t\t//placeholder for mutations`
      }
      for (let i = 0; i < this.userActions.length; i++) {
        str += `\n\t\t${this.userActions[i]} (state) {`;
        str += `\n\t\t\t//placeholder for your mutation`;
        str += `\n\t\t},`;
        if (i === this.userActions.length - 1) { str = str.slice(0, -1) }
      }
      str += `\n\t},`;
      str += `\n\tactions: {`;
      if (!this.userActions.length) {
        str += `\n\t\t\t//placeholder for actions`
      }
      for (let i = 0; i < this.userActions.length; i++) {
        str += `\n\t\t${this.userActions[i]} () {`;
        str += `\n\t\t\tstore.commit('${this.userActions[i]}')`;
        str += `\n\t\t},`;
        if (i === this.userActions.length - 1) { str = str.slice(0, -1) }
      }
      str += `\n\t}`;
      str += '\n})\n';
      str += `\nexport default store;`
      if (this.exportAsTypescript === "on") {
        fs.writeFileSync(path.join(location, "src", "store", "index.ts"), str);
      } else {
        fs.writeFileSync(path.join(location, "src", "store", "index.js"), str);
      }
    },
    // create package.json file
    createPackage(location) {
      let str = `{`;
      str += `\n\t"name": "My-OverVue-Project",`;
      str += `\n\t"version": "0.0.0",`;
      str += `\n\t"scripts": {`;
      str += `\n\t\t"dev": "vite",`;
      if (this.exportAsTypescript === "on") {

        str += `\n\t\t"build": "vue-tsc --noEmit && vite build",`;
        if(this.$store.state.importTest ==='on'){
          str +=`\n\t\t"test:unit": "vue-cli-service test:unit",`
        }

        str += `\n\t\t"typecheck": "vue-tsc --noEmit",`;
        str += `\n\t\t"lint": "eslint . --ext .vue,.js,.jsx,.cjs,.mjs,.ts,.tsx,.cts,.mts --fix --ignore-path .gitignore",`;
      } else {
        str += `\n\t\t"build": "vite build",`;
        if(this.$store.state.importTest ==='on'){
          str +=`\n\t\t"test:unit": "vue-cli-service test:unit",`
        }
        str += `\n\t\t"lint": "eslint . --ext .vue,.js,.jsx,.cjs,.mjs --fix --ignore-path .gitignore",`;
      }
      str += `\n\t\t"preview": "vite preview --port 5050"`;
      str += `\n\t},`;
      str += `\n\t"dependencies": {`;
      str += `\n\t\t"vue": "^3.2.31",`;
      str += `\n\t\t"vue-router": "^4.0.12",`;
      str += `\n\t\t"vuex": "^4.0.2"`;
      str += `,\n\t\t"element-plus": "^2.2.16"`;

      if(this.$store.state.exportOauth ==='on'||this.$store.state.exportOauthGithub ==='on'){
        str += `,\n\t\t "firebase": "^9.6.9"`
      }
      str += `\n\t},`;
      str += `\n\t"devDependencies": {`;
      str += `\n\t\t"@vitejs/plugin-vue": "^2.2.2",`;
      str += `\n\t\t"eslint": "^8.5.0",`;
      str += `\n\t\t"eslint-plugin-vue": "^8.2.0",`;
      str += `\n\t\t"vite": "^2.8.4"`
      if(this.$store.state.importTest ==='on'){
      str+=`,\n\t\t"@babel/core": "^7.12.16",`
      str+=`\n\t\t"@babel/eslint-parser": "^7.12.16",`
      str+=`\n\t\t"@vue/cli-plugin-babel": "~5.0.0",`
      str+=`\n\t\t"@vue/cli-plugin-eslint": "~5.0.0",`
      str+=`\n\t\t"@vue/cli-plugin-unit-jest": "~5.0.0",`
      str+=`\n\t\t"@vue/cli-service": "~5.0.0",`
      str+=`\n\t\t"@vue/test-utils": "^2.0.0-0",`
      str+=`\n\t\t"@vue/vue3-jest": "^27.0.0-alpha.1",`
      str+=`\n\t\t"babel-jest": "^27.0.6",`
      str+=`\n\t\t"jest": "^27.0.5"`
      }
      if (this.exportAsTypescript === "on") {
        str += `,\n\t\t"@rushstack/eslint-patch": "^1.1.0",`
        str += `\n\t\t"@vue/tsconfig": "^0.1.3",`;
        str += `\n\t\t"typescript": "~4.5.5",`;
        str += `\n\t\t"vue-tsc": "^0.31.4",`;
        str += `\n\t\t"@types/node": "^16.11.25",`;
        str += `\n\t\t"@vue/eslint-config-typescript": "^10.0.0"`;
      }
      str += `\n\t}\n}`;
      fs.writeFileSync(path.join(location, "package.json"), str);
    },
    exportFile(data) {
      if (data === undefined) return;
      if (!fs.existsSync(data)) {
        fs.mkdirSync(data);
        fs.mkdirSync(path.join(data, "public"));
        fs.mkdirSync(path.join(data, "src"));
        fs.mkdirSync(path.join(data, "src", "assets"));
        fs.mkdirSync(path.join(data, "src", "components"));
        fs.mkdirSync(path.join(data, "src", "views"));
        fs.mkdirSync(path.join(data, "src", "router"));
        fs.mkdirSync(path.join(data, "src", "store"));
        fs.mkdirSync(path.join(data, "tests"));
        fs.mkdirSync(path.join(data, "tests", "unit"));
      }
      // creating basic boilerplate for vue app
      this.createIndexFile(data);
      this.createMainFile(data);
      this.createViteConfig(data);
      this.createESLintRC(data);
      this.createTSConfig(data);
      this.createTSViteConfig(data);
      this.createTSDeclaration(data);
      this.createPackage(data);
      this.createStore(data);
      this.createFirebaseConfigFile(data);
      this.createOauthFile(data);
      this.createjestConfigFile(data);
      this.createbabelConfigFile(data)
      // exports images to the /assets folder
      // eslint-disable-next-line no-unused-vars
      for (let [routeImage, imageLocation] of Object.entries(this.imagePath)) {
        if (imageLocation !== "") {
          this.createAssetFile(
            path.join(data, "src", "assets", routeImage),
            imageLocation
          );
        }
      }
      // main logic below for creating components
      this.createRouter(data);
      // eslint-disable-next-line no-unused-vars
      for (let componentName in this.componentMap) {
        // if componentName is a route:
        if (componentName !== "App") {
          if (this.$store.state.routes[componentName]) {
            this.createComponentCode(
              path.join(data, "src", "views", componentName),
              componentName,
              this.componentMap
            );
            // if componentName is a just a component
          } else {
            this.createComponentCode(
              path.join(data, "src", "components", componentName),
              componentName,
              this.componentMap
            );
            this.createComponentTestCode(path.join(data, "tests", "unit", componentName + '.spec.js'),
              componentName,
              this.componentMap)
          }
          // if componentName is App
        } else {
          this.createComponentCode(
            path.join(data, "src", componentName),
            componentName,
            this.componentMap
          );
        }
      }
    },
  },
  computed: {
    ...mapState(["componentMap",
    "imagePath",
    "routes",
    "exportAsTypescript",
    "activeComponent",
    "userState",
    "userActions",
    "gridLayout",
    "containerW",
    "containerH"
    ]),
  },
};
</script>

<style scoped>
#export-component-nav-btn {
  margin-bottom: 20px;
}

.menu-btn{
  width: 60%;
  margin: 10px 0px;
  max-height: 55px !important;
}
</style>
