# CD/CI React-native

## Тезисно

1. Предполагается использовать несколько схем для дистрибуции приложения (dev/development, stage/stabe, prod/production)
2. Использование firebase и нескольких GoogleService-Info.plist которые копируются в зависимости от схемы
3. Настройки глобальных переменных которые конфигурируют fastlane передаются из .gitlab-ci.yml
4. Кеширование node_modules + Pods и изменение кеша если призошли изменения в package.json
5. keystore для android создается отдельной job и хранится 30 минут
6. Нотификация в рокет чат канал (нужна настройка)
7. Badge на иконку с версией и этапом разработки (Alpha, Beta)

Для начала нужно создать необходимы Schemes (ios) и productFlavors (Android).
Распространенная практика, - dev, stage, prod
_Полезные ссылки_:
[Handling environment specific configurations in React Native](https://www.bigbinary.com/blog/handling-environment-specific-configurations-in-react-native)
[How to manage staging and production environments in React Native](https://dev.to/calintamas/how-to-manage-staging-and-production-environments-in-a-react-native-app-4naa)
[react-native-config](https://github.com/luggit/react-native-config)

# firebase setup
## iOS
### настройка GoogleService-Info.plist для разных схем

- Создать папку, например ConfigFiles, вставить по папкам названия схем ваши GoogleService-Info.plist-s
<div align="center">
    <img src="/screenshots/config-files.png?raw=true" width="400px"</img> 
</div>

- Добавить Build script phase
<div align="center">
    <img src="/screenshots/copy-google-services-build-phase.png?raw=true" width="400px"</img> 
</div>
<details>
<summary>Copy GoogleService-Info.plist code phase</summary>

```bash
# Type a script or drag a script file from your workspace to insert its path.
# Name of the resource we're selectively copying
GOOGLESERVICE_INFO_PLIST=GoogleService-Info.plist

# Get references to debug, staging and prod versions of the GoogleService-Info.plist
# NOTE: These should only live on the file system and should NOT be part of the target (since we'll be adding them to the target manually)
GOOGLESERVICE_INFO_DEBUG=${PROJECT_DIR}/${TARGET_NAME}/ConfigFiles/dev/${GOOGLESERVICE_INFO_PLIST}
GOOGLESERVICE_INFO_STAGING=${PROJECT_DIR}/${TARGET_NAME}/ConfigFiles/stage/${GOOGLESERVICE_INFO_PLIST}
GOOGLESERVICE_INFO_RELEASE=${PROJECT_DIR}/${TARGET_NAME}/ConfigFiles/prod/${GOOGLESERVICE_INFO_PLIST}

# Make sure the debug version of GoogleService-Info.plist exists
echo "Looking for ${GOOGLESERVICE_INFO_PLIST} in ${GOOGLESERVICE_INFO_DEBUG}"
if [ ! -f $GOOGLESERVICE_INFO_DEBUG ]
then
    echo "No Debug GoogleService-Info.plist found. Please ensure it's in the proper directory."
    exit 1
fi

# Make sure the staging version of GoogleService-Info.plist exists
echo "Looking for ${GOOGLESERVICE_INFO_PLIST} in ${GOOGLESERVICE_INFO_STAGING}"
if [ ! -f $GOOGLESERVICE_INFO_PROD ]
then
    echo "No Staging GoogleService-Info.plist found. Please ensure it's in the proper directory."
    exit 1
fi

# Make sure the release version of GoogleService-Info.plist exists
echo "Looking for ${GOOGLESERVICE_INFO_PLIST} in ${GOOGLESERVICE_INFO_RELEASE}"
if [ ! -f $GOOGLESERVICE_INFO_RELEASE ]
then
    echo "No Release GoogleService-Info.plist found. Please ensure it's in the proper directory."
    exit 1
fi

# Get a reference to the destination location for the GoogleService-Info.plist
PLIST_DESTINATION=${BUILT_PRODUCTS_DIR}/${PRODUCT_NAME}.app
echo "Will copy ${GOOGLESERVICE_INFO_PLIST} to final destination: ${PLIST_DESTINATION}"

# Copy over the prod GoogleService-Info.plist for Release builds
if [ "${CONFIGURATION}" == "ReleaseDev" ]
then
    echo "Using ${GOOGLESERVICE_INFO_DEBUG}"
    cp "${GOOGLESERVICE_INFO_DEBUG}" "${PLIST_DESTINATION}"
fi

if [ "${CONFIGURATION}" == "ReleaseStage" ]
then
    echo "Using ${GOOGLESERVICE_INFO_STAGING}"
    cp "${GOOGLESERVICE_INFO_STAGING}" "${PLIST_DESTINATION}"
fi

if [ "${CONFIGURATION}" == "ReleaseProd" ]
then
    echo "Using ${GOOGLESERVICE_INFO_RELEASE}"
    cp "${GOOGLESERVICE_INFO_RELEASE}" "${PLIST_DESTINATION}"
fi

if [ "${CONFIGURATION}" == "DebugDev" ]
then
    echo "Using ${GOOGLESERVICE_INFO_DEBUG}"
    cp "${GOOGLESERVICE_INFO_DEBUG}" "${PLIST_DESTINATION}"
fi

if [ "${CONFIGURATION}" == "DebugStage" ]
then
    echo "Using ${GOOGLESERVICE_INFO_STAGING}"
    cp "${GOOGLESERVICE_INFO_STAGING}" "${PLIST_DESTINATION}"
fi

if [ "${CONFIGURATION}" == "DebugProd" ]
then
    echo "Using ${GOOGLESERVICE_INFO_RELEASE}"
    cp "${GOOGLESERVICE_INFO_RELEASE}" "${PLIST_DESTINATION}"
fi
```
</details>

## Android

Переместить google-services.json в папку с нзванием product Flavor
<div align="center">
    <img src="/screenshots/copy-google-services-build-phase-android.png?raw=true" width="400px"</img> 
</div>

все возможные задания для gradle можно посмореть в Gradle панели Android Studio либо в папке android набрать __./gradlew tasks__

<details>
<summary>.gitlab-ci.yml</summary>

```yml
variables:
  GIT_SSL_NO_VERIFY: "true"
  LC_ALL: "en_US.UTF-8"
  LANG: "en_US.UTF-8"
  GET_SOURCES_ATTEMPTS: "10"

cache: &global_cache
  key:
    files:
      - Gemfile.lock
      - package.json
    prefix: $CI_PROJECT_NAME
  paths:
    - node_modules/
  policy: pull

stages:
  - lint
  - build
  - deploy

prepare:
  stage: .pre
  cache:
    <<: *global_cache
    policy: pull-push
  tags:
    - rocket-ios
  allow_failure:
    exit_codes: 10
  script:
    - |
      if [[ -d node_modules ]]; then
        exit 10
      fi
    - yarn install --frozen-lockfile
    - bundle install
    - fastlane install_plugins --verbose

create_android_keystore:
  stage: .pre
  tags:
    - rocket-ios
  script:
    - echo $KEYSTORE | base64 -d > android/app/my.keystore
    - echo "keystorePath=my.keystore" > android/app/signing.properties
    - echo "keystorePassword=$KEYSTORE_PASSWORD" >> android/app/signing.properties
    - echo "keyAlias=$KEY_ALIAS" >> android/app/signing.properties
    - echo "keyPassword=$KEY_PASSWORD" >> android/app/signing.properties
  artifacts:
    paths:
      - android/app/my.keystore
      - android/app/signing.properties
    expire_in: 30 mins

eslint:
  stage: lint
  tags:
    - rocket-ios
  only:
    - dev
    - merge_requests
  script:
    - npm run lint

.android:
  image: openjdk:8-jdk
  dependencies:
    - create_android_keystore
  cache:
    key: ${CI_COMMIT_REF_SLUG}
    paths:
      - .gradle/wrapper
      - .gradle/caches

.build_script:
  script:
    - bundle exec fastlane ios build

.deploy_script:
  script:
    - bundle exec fastlane ios upload_appcenter

.dev:
  variables:
    CI_COMMIT_BRANCH: "dev"
    SCHEME: "Dev"
    CONFIGURATION: "ReleaseDev"
    DIST_TYPE: "enterprise"
    GRADLE_APK_OUTPUT_PATH: "android/app/build/outputs/apk/devPlay/release/app-dev-play-release.apk"
  environment:
    name: dev
  only:
    - dev
  tags:
    - rocket-ios
  artifacts:
    paths:
      - "$GRADLE_APK_OUTPUT_PATH"
      - "ReleaseDev/RocketChat_Dev.ipa"

ios:build:dev:
  stage: build
  needs: ["eslint"]
  cache:
    <<: *global_cache
  extends:
    - .dev
    - .build_script

ios:deploy:dev:
  stage: deploy
  needs: ["ios:build:dev"]
  dependencies:
    - ios:build:dev
  before_script: []
  extends:
    - .dev
    - .deploy_script
  cache: []

android:build:dev:
  stage: build
  needs: ["eslint", "create_android_keystore"]
  cache:
    <<: *global_cache
  extends:
    - .dev
    - .android
  script:
    - bundle exec fastlane android build

android:deploy:dev:
  stage: deploy
  needs: ["android:build:dev", "create_android_keystore"]
  dependencies:
    - android:build:dev
  before_script: []
  extends:
    - .dev
    - .android
  cache: []
  script:
    - bundle exec fastlane android upload_appcenter

.stable:
  variables:
    CI_COMMIT_BRANCH: "stable"
    SCHEME: "Stage"
    CONFIGURATION: "ReleaseStage"
    DIST_TYPE: "enterprise"
    GRADLE_APK_OUTPUT_PATH: "android/app/build/outputs/apk/stagePlay/release/app-stage-play-release.apk"
  environment:
    name: staging
  only:
    - stable
  tags:
    - rocket-ios
  artifacts:
    name: "$CONFIGURATION"
    paths:
      - "$CONFIGURATION/"
      - "$GRADLE_APK_OUTPUT_PATH"

ios:build:stable:
  stage: build
  needs: ["eslint"]
  cache:
    <<: *global_cache
  extends:
    - .stable
    - .build_script

ios:deploy:stable:
  stage: deploy
  needs: ["ios:build:stable"]
  before_script: []
  extends:
    - .stable
    - .deploy_script
  cache: []

android:build:stable:
  stage: build
  needs: ["eslint", "create_android_keystore"]
  cache:
    <<: *global_cache
  extends:
    - .stable
    - .android
  script:
    - bundle exec fastlane android build

android:deploy:stable:
  stage: deploy
  needs: ["android:build:stable", "create_android_keystore"]
  dependencies:
    - android:build:stable
  before_script: []
  extends:
    - .stable
  cache: []
  script:
    - bundle exec fastlane android upload_appcenter

.release:
  variables:
    CI_COMMIT_BRANCH: "main"
    IS_RELEASE: "true"
    SCHEME: "Prod"
    CONFIGURATION: "ReleaseProd"
    DIST_TYPE: "enterprise"
    GRADLE_APK_OUTPUT_PATH: "android/app/build/outputs/apk/releasePlay/release/app-release-play-release.apk"
  environment:
    name: production
  only:
    - main
  tags:
    - rocket-ios
  artifacts:
    name: "$CONFIGURATION"
    paths:
      - "$CONFIGURATION/"
      - "$GRADLE_APK_OUTPUT_PATH"

ios:build:release:
  stage: build
  needs: ["eslint"]
  cache:
    <<: *global_cache
  extends:
    - .release
    - .build_script

ios:deploy:release:
  stage: deploy
  needs: ["ios:build:release"]
  before_script: []
  extends:
    - .release
    - .deploy_script
  cache: []

android:build:release:
  stage: build
  needs: ["eslint", "create_android_keystore"]
  cache:
    <<: *global_cache
  extends:
    - .release
    - .android
  script:
    - bundle exec fastlane android build

android:deploy:release:
  stage: deploy
  needs: ["android:build:release", "create_android_keystore"]
  dependencies:
    - android:build:release
  before_script: []
  extends:
    - .release
  cache: []
  script:
    - bundle exec fastlane android upload_appcenter

```
</details>

<details>
<summary>Fastfile</summary>

```ruby
before_all do
  update_fastlane
  changelog
end

# ENV["SCHEME"] # Dev, Stage, Prod
# ENV["CONFIGURATION"] # ReleaseDev, ReleaseStage, ReleaseProd
# ENV["IS_RELEASE"]
notes = nil

desc "Create changelog"
lane :changelog do
  analyze_commits(match: "v*")
  notes = "No changelog given"
  begin 
    notes = conventional_changelog(
      order: ["feat", "fix", "docs", "refactor", "chore" "ci"],
      commit_url: ENV["COMMIT_URL"]
    )
  rescue => ex
    UI.error(ex)
  end
  notes
end

desc "Rocket chat notify"
lane :changelog do
  rocket_chat(
    message: "App successfully released!",
    channel: "#test_notify",  # Optional, by default will post to the default channel configured for the POST URL.
    success: true,        # Optional, defaults to true.
    payload: {            # Optional, lets you specify any number of your own Rocket.Chat attachments.
      'Build Date' => Time.new.to_s,
      'Built by' => 'CI gitlab',
    },
    default_payloads: [:git_branch, :git_author, :lane, :test_result, :last_git_commit_message], 
    # Optional, lets you specify a whitelist of default payloads to include. 
    # Pass an empty array to suppress all the default payloads. 
    # Don't add this key, or pass nil, if you want all the default payloads. 
    # The available default payloads are: `lane`, `test_result`, `git_branch`, `git_author`, `last_git_commit_message`.
    attachment_properties: { 
      # Optional, lets you specify any other properties available for attachments. This hash is deep merged with the existing properties set using the other properties above. This allows your own fields properties to be appended to the existing fields that were created using the `payload` property for instance.
      thumb_url: 'https://about.gitlab.com/images/press/logo/png/gitlab-icon-rgb.png',
      fields: [{
        title: 'My Field',
        value: 'My Value'
      }]
    }
  )
end

# Добавляем badge на иконку в зависимости от схемы
# в данный момент работает только для ios
desc "add badge"
  lane :set_badge do
    build_number = get_build_number(xcodeproj: "./ios/RocketChatRN.xcodeproj")
    package = load_json(json_path: "./package.json")
    # если не релиз, то добавялем
    if !ENV["IS_RELEASE"]
      add_badge(shield: "#{package["version"]}-(#{build_number})-green", alpha: ENV["SCHEME"] == "Dev" ? true : false)  
    end
end

# Сперва надо увеличить версию в package.json (npm bump --minor --major --patch) 
# build number(iOS)/versionCode(android) не трогаем совсем, только версию
desc "Set new version"
  lane :set_new_version do
    # changelog
    package = load_json(json_path: "./package.json")
    increment_version_number( version_number: package["version"], xcodeproj: "./ios/RocketChatRN.xcodeproj")
    increment_build_number(xcodeproj: "./ios/RocketChatRN.xcodeproj")

    android_set_version_name( version_name: package["version"], gradle_file: "./android/app/build.gradle")
    android_set_version_code( gradle_file: "./android/app/build.gradle")
  end

platform :ios do

  # разблокируем keychain macos чтобы использовать сертификаты для подписи
  desc 'Unlock keychain'
  private_lane :unlock_key do
    unlock_keychain(
      path: ENV["KEYCHAIN_PATH"],
      password: ENV["KEYCHAIN_PASSWORD"]
    )
  end


  desc "Get ios certificates for build environment"
  private_lane :get_right_certs do
    match( type: ENV["DIST_TYPE"], readonly: is_ci)
  end

  # i think need some refactoring/rebuild
  desc "push to git"
  lane :push_to_git do |options|
    package = load_json(json_path: "./package.json")
    build_number = get_build_number(xcodeproj: "./ios/RocketChatRN.xcodeproj")
    git_add(path: ["./ios/RocketChatRN.xcodeproj/project.pbxproj", "./ios/RocketChatRN/Info.plist", "./android/app/build.gradle"])
    if ENV["IS_RELEASE"]
      add_git_tag(tag: "v#{package["version"]}")
      git_commit(path: ["./ios/RocketChatRN.xcodeproj/project.pbxproj", "./ios/RocketChatRN/Info.plist", "./android/app/build.gradle"], message: "New release #{package["version"]}(#{build_number})")
    else
      git_commit(path: ["./ios/RocketChatRN.xcodeproj/project.pbxproj", "./ios/RocketChatRN/Info.plist", "./android/app/build.gradle"], message: "New build #{package["version"]}(#{build_number})")
    end
    push_to_git_remote(
      local_branch: "HEAD",
      remote_branch: "dev", 
      force: true,    
    )
  end

  # Линия билда апки
  desc 'Build app'
  lane :build do
    UI.important "configuration: #{ENV["CONFIGURATION"]} scheme: #{ENV["SCHEME"]}"
    
    unlock_key
    get_right_certs
    set_new_version
    set_badge

    build_app(
      workspace: "./ios/RocketChatRN.xcworkspace",
      configuration: ENV["CONFIGURATION"],
      scheme: "RocketChat#{ENV["SCHEME"]}",
      silent: true,
      clean: true,
      export_method: "enterprise",
      export_xcargs: "-allowProvisioningUpdates",
      xcargs: "-allowProvisioningUpdates",
      output_directory: "#{ENV["CONFIGURATION"]}",
      output_name: "RocketChat_#{ENV["SCHEME"]}.ipa",
    )
    # push_to_git
  end

  desc "Upload to app center"
  lane :upload_appcenter do
    appcenter_upload(
      app_name: "RocketChat-#{ENV["SCHEME"]}",
      file: "./#{ENV["CONFIGURATION"]}/RocketChat_#{ENV["SCHEME"]}.ipa",
      destinations: '*',
      notify_testers: true,
      release_notes: notes
    )
  end

  # Линии который можно дернуть чтобы собрать определенную схему апки
  desc 'Build Dev configuration'
  lane :release_dev do
    ENV["SCHEME"] = "Dev"
    ENV["CONFIGURATION"] = "ReleaseDev"
    build
  end

  desc 'Build Stage configuration'
  lane :release_stage do
    ENV["SCHEME"] = "Stage"
    ENV["CONFIGURATION"] = "ReleaseStage"
    build
  end

  desc 'Build Prod configuration'
  lane :release_prod do
    ENV["SCHEME"] = "Prod"
    ENV["CONFIGURATION"] = "ReleaseProd"
    build
  end

end

platform :android do

  desc "Build"
  lane :build do

    set_new_version

    gradle(
      task: 'assemble',
      flavor: "#{ENV["SCHEME"]}",
      project_dir: "./android",
      build_type: "Release"
    )

  end

  desc "Upload to app center"
  lane :upload_appcenter do
    # notes = changelog
    appcenter_upload(
      app_name: "RocketChat-#{ENV["SCHEME"]}-1",
      file: ENV["GRADLE_APK_OUTPUT_PATH"],
      destinations: '*',
      notify_testers: true,
      release_notes: notes
    )
  end
end

```
</details>

### гайд может дополняться/изменяться
по возможным вопросам __a-pavlov@ocs.ru__ либо в __RocketChat__ Pavlov Aleksey

