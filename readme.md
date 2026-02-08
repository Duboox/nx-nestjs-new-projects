"build": {
      "executor": "@nx/webpack:webpack",
      "outputs": [
        "{options.outputPath}"
      ],
      "defaultConfiguration": "production",
      "options": {
        "target": "node",
        "compiler": "tsc",
        "outputPath": "dist/apps/uploads",
        "main": "apps/uploads/src/main.ts",
        "tsConfig": "apps/uploads/tsconfig.app.json",
        "isolatedConfig": true,
        "webpackConfig": "apps/uploads/webpack.config.js",
        "generatePackageJson": true,
        "assets": [
          {
            "glob": "**/*",
            "input": "libs/common/src/i18n/translations",
            "output": "./i18n/translations"
          }
        ]
      },
      "configurations": {
        "development": {
          "mode": "development"
        },
        "production": {
          "mode": "production"
        }
      }
    },

    "serve": {
      "continuous": true,
      "executor": "@nx/js:node",
      "defaultConfiguration": "development",
      "dependsOn": [
        "build"
      ],
      "options": {
        "buildTarget": "uploads:build",
        "runBuildTargetDependencies": false,
        "watch": true
      },
      "configurations": {
        "development": {
          "buildTarget": "uploads:build:development"
        },
        "production": {
          "buildTarget": "uploads:build:production"
        }
      }
    },




    tsconfig.app.json
    {
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "outDir": "../../dist/out-tsc",
    "module": "commonjs",
    "types": [
      "node"
    ],
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true,
    "target": "es2021"
  },
  "include": [
    "src/**/*.ts",
  ],
  "exclude": [
    "jest.config.ts",
    "src/**/*.spec.ts",
    "src/**/*.test.ts"
  ]
}


webpack.config.js
const { composePlugins, withNx } = require('@nx/webpack');

module.exports = composePlugins(withNx(), (config) => {
  return config;
});


docker compose

  # ***** UPLOADS SERVICE *****
  uploads:
    build:
      context: .
      dockerfile: ./apps/uploads/Dockerfile
      target: development
    environment:
    command: yarn nx serve uploads --inspect=false
    env_file:
      - ./apps/uploads/.env
    networks:
      - internal-network #Port 3070
    ports:
      - "3070:3070" #TODO: DELETE THIS
    volumes:
      - .:/usr/src/app
      - /usr/src/app/node_modules # use dockerfile built node_modules
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:3070/health || exit 1"]
      interval: 60s
      timeout: 5s
      retries: 5
      start_period: 15s
